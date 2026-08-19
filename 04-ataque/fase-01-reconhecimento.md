# Fase 1: Reconhecimento

> Operação Phantom Helper — Milvus Summit 2026
> MITRE ATT&CK: TA0043 — Reconnaissance

---

## Objetivo da Fase

Mapear a superfície de ataque da TechAssist S.A. e identificar o sistema de help desk como vetor de entrada. O reconhecimento é a fase mais crítica de um ataque — um atacante paciente e metódico ganha aqui a vantagem estratégica que decide o sucesso ou fracasso da operação.

**Duração estimada**: 2-3 dias (ataque real) | 30 minutos (laboratório)
**Nível de ruído**: Baixíssimo — a maioria das técnicas usa fontes abertas
**Detecção**: Improvável nesta fase

---

## Técnicas MITRE ATT&CK Utilizadas

| Tática | ID | Técnica |
|--------|-----|---------|
| Reconnaissance | T1593.001 | Search Open Websites/Domains: Social Media |
| Reconnaissance | T1593.002 | Search Open Websites/Domains: Search Engines |
| Reconnaissance | T1596.005 | Search Open Technical Databases: Scan Databases |
| Reconnaissance | T1590 | Gather Victim Network Information |
| Reconnaissance | T1591 | Gather Victim Org Information |

---

## 1.1 OSINT na TechAssist S.A.

### 1.1.1 Pesquisa no LinkedIn

O LinkedIn é a maior base de inteligência competitiva corporativa disponível publicamente. Para um atacante, é o primeiro passo.

```bash
# Usando theHarvester com suporte a LinkedIn
theHarvester -d techassist.com.br -b google,linkedin,bing,twitter -l 500 -f recon/techassist_harvest.html

# Output esperado:
# [*] Emails found:
#     carla.santos@techassist.com.br
#     rafael.gomes@techassist.com.br
#     joao.ceo@techassist.com.br
#     helpdesk@techassist.com.br
#     ti@techassist.com.br
#
# [*] Hosts found:
#     helpdesk.techassist.com.br
#     mail.techassist.com.br
#     vpn.techassist.com.br
#     wiki.techassist.com.br
```

**O que foi aprendido com LinkedIn:**
- TechAssist tem ~250 funcionários
- Escritório na Avenida Paulista, 14° andar
- Stack tecnológica exposta nos perfis de desenvolvedores: "Python", "FastAPI", "Ollama", "Node.js"
- Clientes mencionados em posts: Construtora Horizonte, Clínica Vida, escritórios de advocacia
- Job posting recente: "Analista de Suporte — experiência com plataforma de help desk IA"
- Funcionária Carla Santos: "Analista de TI há 3 anos, help desk desde 2022"

> **Insight crítico**: O perfil de um engenheiro de backend mencionava "implementamos ARIA, nosso assistente de IA com integração LDAP e PostgreSQL". Informação pública. Resultado: conhecimento do nome interno do sistema, linguagem (Python), e integrações.

---

### 1.1.2 Reconhecimento de Subdomínios

```bash
# Enumeração passiva de subdomínios
subfinder -d techassist.com.br -all -o recon/subdomains.txt

# Output:
# helpdesk.techassist.com.br
# mail.techassist.com.br
# vpn.techassist.com.br
# portal.techassist.com.br
# api.techassist.com.br

# Verificação de IPs dos subdomínios
cat recon/subdomains.txt | dnsx -a -resp -o recon/subdomains_ips.txt

# Força bruta de subdomínios adicionais
gobuster dns -d techassist.com.br \
    -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt \
    -o recon/gobuster_dns.txt
```

**Subdomínios encontrados e mapeados:**

| Subdomínio | IP | Serviço Inferido |
|-----------|-----|-----------------|
| helpdesk.techassist.com.br | 10.0.0.5 (via reverse proxy) | Sistema de help desk (ARIA) |
| mail.techassist.com.br | 10.0.0.15 | Servidor de email |
| vpn.techassist.com.br | 10.0.0.20 | VPN corporativa |
| api.techassist.com.br | 10.0.0.5 (mesmo proxy) | API pública |

---

### 1.1.3 Busca no Shodan

```bash
# Busca por sistemas vulneráveis associados ao domínio
pip install shodan
shodan search "techassist.com.br" --fields ip_str,port,hostnames,org

# Busca por tecnologias específicas encontradas via OSINT
shodan search "FastAPI help desk Brazil" --fields ip_str,port,data | head -20

# Busca por certificado SSL do domínio
shodan search "ssl.cert.subject.cn:techassist.com.br" --fields ip_str,port,hostnames

# Busca por banners específicos
shodan search "ARIA Help Desk API" --fields ip_str,port,data
```

**Descoberta crítica no Shodan:**

```
IP: [IP_PUBLICO_TECHASSIST]
Port: 443
Hostnames: helpdesk.techassist.com.br

HTTP/1.1 200 OK
Server: nginx/1.24.0
X-Powered-By: ARIA-Engine/2.3.1
Content-Type: application/json

{"service": "ARIA Help Desk API", "version": "2.3.1", "status": "online"}
```

> **Descoberta crítica**: O header `X-Powered-By: ARIA-Engine/2.3.1` expõe o nome e versão exatos do sistema. Uma busca no GitHub por "ARIA-Engine" revela repositórios públicos similares — e seus endpoints documentados.

---

## 1.2 Scan de Portas e Serviços

```bash
# Criar diretório para resultados
mkdir -p recon

# Scan rápido de portas abertas (top 1000)
nmap -F --open -oG recon/quickscan.gnmap helpdesk.techassist.com.br

# Scan detalhado com detecção de serviços e versões
nmap -sV -sC -p- --open \
    -T4 \
    --script=http-headers,http-title,ssl-cert \
    -oA recon/techassist_full \
    helpdesk.techassist.com.br

# Output relevante:
# PORT    STATE SERVICE  VERSION
# 80/tcp  open  http     nginx 1.24.0 (redirect to 443)
# 443/tcp open  ssl/http nginx 1.24.0
# |_http-title: TechAssist Help Desk - ARIA
# | ssl-cert:
# |   Subject: CN=helpdesk.techassist.com.br
# |   Not After: 2025-12-31
# |_http-server-header: nginx/1.24.0
```

---

## 1.3 Fingerprinting da Aplicação Web

```bash
# Identificação de tecnologias
whatweb -v https://helpdesk.techassist.com.br

# Output:
# https://helpdesk.techassist.com.br [200 OK]
# HTTPServer[nginx/1.24.0]
# X-Powered-By[ARIA-Engine/2.3.1]
# Title[TechAssist Help Desk - Powered by ARIA]
# Script[text/javascript][/static/js/main.b4f2d891.js]
# React[detectado via bundle]
# Bootstrap[5.3]

# Análise do JavaScript principal (buscando endpoints da API)
curl -s https://helpdesk.techassist.com.br/static/js/main.b4f2d891.js | \
    grep -oE '"(/api/[^"]+)"' | sort -u

# Resultado:
# "/api/v1/chat"
# "/api/v1/tickets"
# "/api/v1/info"
# "/api/v1/tickets/search"
# "/health"
```

---

## 1.4 Enumeração de Endpoints da API

```bash
# Enumeração de diretórios e endpoints
gobuster dir \
    -u https://helpdesk.techassist.com.br \
    -w /usr/share/wordlists/SecLists/Discovery/Web-Content/api/api-endpoints.txt \
    -x json,txt \
    -o recon/gobuster_api.txt \
    -b 404,403

# Resultado:
# /api/v1/chat         (Status: 405) [Método: precisa POST]
# /api/v1/info         (Status: 200) [Endpoint aberto!]
# /api/v1/tickets      (Status: 401) [Requer auth]
# /health              (Status: 200) [Expõe informações do sistema]
# /docs                (Status: 200) [DOCUMENTAÇÃO SWAGGER EXPOSTA!]

# Acessar a documentação Swagger automaticamente gerada pelo FastAPI
curl -s https://helpdesk.techassist.com.br/docs | grep -A5 "title"

# Obter o schema completo da API via OpenAPI
curl -s https://helpdesk.techassist.com.br/openapi.json | python3 -m json.tool
```

**Descoberta explosiva: Swagger UI exposta**

```json
{
  "openapi": "3.0.2",
  "info": {
    "title": "ARIA Help Desk API",
    "version": "2.3.1"
  },
  "paths": {
    "/api/v1/chat": {
      "post": {
        "summary": "Chat",
        "requestBody": {
          "content": {
            "application/json": {
              "schema": {
                "properties": {
                  "user_id": {"type": "string"},
                  "client_company": {"type": "string"},
                  "message": {"type": "string"},
                  "category": {"type": "string"}
                }
              }
            }
          }
        }
      }
    },
    "/api/v1/info": {
      "get": {
        "summary": "System Info",
        "description": "Returns system information and available tools"
      }
    }
  }
}
```

> **Descoberta crítica**: A documentação Swagger estava pública. O endpoint `/api/v1/info` retorna a lista completa de ferramentas disponíveis para a IA. O endpoint `/api/v1/chat` aceita qualquer mensagem sem autenticação.

---

## 1.5 Consulta ao Endpoint de Informações

```bash
# Consultando /api/v1/info sem autenticação
curl -s https://helpdesk.techassist.com.br/api/v1/info | python3 -m json.tool
```

**Resposta do sistema:**

```json
{
  "version": "2.3.1",
  "model": "llama3:8b",
  "tools_available": [
    "ad_query",
    "reset_password",
    "create_temp_user",
    "get_user_tickets",
    "search_tickets",
    "get_client_data",
    "send_email",
    "list_shares"
  ],
  "integrations": [
    "active_directory",
    "postgresql",
    "smtp",
    "smb"
  ]
}
```

**BINGO.** O sistema expôs publicamente:
1. O nome e versão do modelo de LLM em uso
2. Todas as ferramentas disponíveis para o agente de IA
3. Todos os sistemas integrados

Isso elimina horas de reconhecimento — Phantom sabe exatamente o que pode explorar.

---

## 1.6 Teste de Conectividade com o Endpoint de Chat

```bash
# Teste básico para verificar se o chat aceita requisições sem autenticação
curl -s -X POST \
    https://helpdesk.techassist.com.br/api/v1/chat \
    -H "Content-Type: application/json" \
    -d '{
        "user_id": "test.user",
        "client_company": "Test Corp",
        "message": "Olá, como posso abrir um ticket?",
        "category": "geral"
    }' | python3 -m json.tool
```

**Resposta:**

```json
{
  "ticket_id": "TKT20250407083145",
  "user_id": "test.user",
  "response": "Olá! Fico feliz em ajudar. Para abrir um ticket, você pode simplesmente descrever seu problema aqui...",
  "timestamp": "2025-04-07T08:31:45.221Z"
}
```

**Confirmado**: O endpoint `/api/v1/chat` aceita requisições **sem qualquer autenticação**, com **qualquer `user_id`** e **qualquer `client_company`**. Qualquer pessoa no mundo pode enviar mensagens para a IA.

---

## 1.7 Análise de Certificado SSL e Infraestrutura

```bash
# Análise detalhada do certificado SSL
echo | openssl s_client -connect helpdesk.techassist.com.br:443 2>/dev/null | \
    openssl x509 -noout -text | grep -E "Subject:|Issuer:|Not|DNS:"

# Busca por outros domínios no certificado (SAN)
echo | openssl s_client -connect helpdesk.techassist.com.br:443 2>/dev/null | \
    openssl x509 -noout -text | grep "DNS:"

# Resultado:
# DNS: helpdesk.techassist.com.br
# DNS: mail.techassist.com.br
# DNS: portal.techassist.com.br
# DNS: vpn.techassist.com.br
# DNS: wiki.techassist.com.br

# Verificar WHOIS para informações de registrant
whois techassist.com.br | grep -E "owner:|e-mail:|phone:|nic-hdl:"
```

---

## 1.8 Mapa de Superfície de Ataque — Resumo do Reconhecimento

Após o reconhecimento, Phantom tinha o seguinte mapa:

```
┌─────────────────────────────────────────────────────────────────────┐
│  MAPA DE SUPERFÍCIE DE ATAQUE — TechAssist S.A.                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  VETORES DE ENTRADA IDENTIFICADOS:                                   │
│                                                                      │
│  [ALTO RISCO] Chat público sem autenticação                          │
│               URL: https://helpdesk.techassist.com.br/api/v1/chat   │
│               Aceita: qualquer usuário, qualquer empresa             │
│               IA disponível: llama3:8b com 8 ferramentas             │
│                                                                      │
│  [MÉDIO RISCO] Swagger UI exposto                                    │
│               URL: https://helpdesk.techassist.com.br/docs           │
│               Exposição: estrutura completa da API                   │
│                                                                      │
│  [MÉDIO RISCO] Endpoint /api/v1/info público                         │
│               Exposição: lista de ferramentas + integrações          │
│                                                                      │
│  FERRAMENTAS DA IA IDENTIFICADAS:                                    │
│  - ad_query() → Active Directory (CRÍTICO)                           │
│  - reset_password() → Modificação de AD (CRÍTICO)                   │
│  - get_client_data() → PostgreSQL (CRÍTICO)                          │
│  - send_email() → SMTP (ALTO)                                        │
│  - list_shares() → SMB (ALTO)                                        │
│                                                                      │
│  CREDENCIAIS ALVO PRIORIZADOS:                                       │
│  - backup.admin (senha antiga — 500+ dias)                           │
│  - svc.helpdesk (conta de serviço da IA)                             │
│                                                                      │
│  DADOS ALVO IDENTIFICADOS:                                           │
│  - PostgreSQL: clientesdb (15k registros estimados via OSINT)        │
│  - SMB: Contratos, Confidencial (inferido via LinkedIn)              │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Ferramentas Utilizadas

| Ferramenta | Versão | Propósito | Disponível no Kali |
|------------|--------|-----------|-------------------|
| theHarvester | 4.x | OSINT emails/hosts | Sim (nativo) |
| subfinder | 2.6 | Subdomínios | `apt install subfinder` |
| gobuster | 3.x | Força bruta de diretórios | Sim (nativo) |
| nmap | 7.94 | Scan de portas | Sim (nativo) |
| whatweb | 0.5 | Fingerprint web | Sim (nativo) |
| shodan-cli | 1.x | Busca em Shodan | `pip install shodan` |
| dnsx | 1.x | Resolução DNS em massa | `apt install dnsx` |
| curl | 8.x | Testes de API | Sim (nativo) |
| openssl | 3.x | Análise SSL | Sim (nativo) |

---

## Próxima Fase

Com o mapa de superfície de ataque completo, o atacante tem tudo que precisa para iniciar a exploração. O vetor escolhido é o mais silencioso e efetivo: o próprio chat de help desk, via **Prompt Injection**.

**Próximo passo**: `fase-02-acesso-inicial.md`

---

*Operação Phantom Helper — Milvus Summit 2026*
*Almir Meira Alves — Meira e Sousa Ltda — Educa com Talento*
