# Operação Phantom Helper — Quando a IA do Help Desk Vira Arma

![Hero Banner](assets/hero-banner.svg)

---

[![Milvus Summit 2026](https://img.shields.io/badge/🎯_Milvus_Summit-2026-6f42c1?style=for-the-badge)](https://milvus.com.br)
[![OWASP LLM Top 10](https://img.shields.io/badge/OWASP_LLM-Top_10-orange?style=for-the-badge&logo=owasp)](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
[![MITRE ATT&CK](https://img.shields.io/badge/MITRE_ATT%26CK-v14-red?style=for-the-badge)](https://attack.mitre.org)
[![MITRE ATLAS](https://img.shields.io/badge/MITRE_ATLAS-v3-darkred?style=for-the-badge)](https://atlas.mitre.org)
[![Técnica Principal](https://img.shields.io/badge/Técnica-Prompt_Injection-critical?style=for-the-badge)](https://owasp.org)
[![Idioma](https://img.shields.io/badge/Idioma-PT--BR-009c3b?style=for-the-badge)](.)
[![LLM](https://img.shields.io/badge/LLM-Llama_3_8B-blueviolet?style=for-the-badge)](https://ollama.com)
[![Framework](https://img.shields.io/badge/Runtime-FastAPI_+_Ollama-teal?style=for-the-badge)](.)
[![Nível](https://img.shields.io/badge/Nível-Técnico_Avançado-blue?style=for-the-badge)](.)
[![Impacto](https://img.shields.io/badge/Impacto-R$_4.080.000-ff0000?style=for-the-badge)](.)
[![NIST](https://img.shields.io/badge/IR-NIST_SP_800--61-informational?style=for-the-badge)](https://csrc.nist.gov)

---

> **Exercício de cibersegurança para o Milvus Summit 2026**
> 19 de agosto de 2026 — São Paulo, SP

---

## Descrição Geral

A **Operação Phantom Helper** é um exercício técnico de cibersegurança que demonstra como sistemas de help desk com Inteligência Artificial podem se tornar vetores de ataque sofisticados quando implementados sem os controles de segurança adequados.

O exercício foi desenvolvido para a palestra técnica no **Milvus Summit 2026**, evento da Milvus Tecnologia — empresa referência em software de help desk com IA no Brasil. A proposta é demonstrar, de forma prática e envolvente, os riscos reais que organizações enfrentam ao integrar IA com sistemas corporativos críticos sem uma postura de segurança robusta.

### O Que Este Repositório Contém

Este repositório documenta um exercício completo de Red Team contra um sistema de help desk com IA, cobrindo:

- **Storytelling cinematográfico** do incidente para engajamento da audiência
- **5 fases de ataque** documentadas com comandos reais e explicações técnicas
- **Mapeamento MITRE ATT&CK** completo de todas as técnicas utilizadas
- **Mapeamento OWASP LLM Top 10** das vulnerabilidades exploradas
- **Playbook de resposta a incidente** seguindo NIST SP 800-61
- **Guia de prevenção** em camadas (Defense in Depth)
- **Análise de impacto** quantificado no negócio

---

## Estrutura do Repositório

```
OperacaoPhantomHelper/
├── README.md                          # Este arquivo
│
├── 01-storytelling/
│   └── historia.md                    # Narrativa cinematográfica do incidente
│
├── 04-ataque/
│   ├── fase-01-reconhecimento.md      # OSINT e mapeamento da superfície de ataque
│   ├── fase-02-acesso-inicial.md      # Prompt Injection — acesso inicial via IA
│   ├── fase-03-comprometimento-ia.md  # Exploração das ferramentas do agente de IA
│   ├── fase-04-lateralizacao-persistencia.md  # Movimento lateral e backdoors
│   └── fase-05-exfiltracao-impacto.md # Exfiltração de dados e ransomware
│
├── 05-impacto/
│   └── impacto-negocio.md             # Análise quantificada do impacto no negócio
│
├── 06-mitre-attack/
│   └── mapeamento-mitre.md            # MITRE ATT&CK + OWASP LLM Top 10
│
├── 07-resposta-incidente/
│   └── resposta-incidente.md          # Playbook NIST SP 800-61 completo
│
└── 08-prevencao/
    └── acoes-preventivas.md           # Defense in Depth para sistemas de IA
```

---

## Técnica Principal: Prompt Injection (OWASP LLM01)

O ataque central deste exercício é o **Prompt Injection**, classificado como LLM01 no OWASP LLM Top 10. Esta vulnerabilidade permite que um atacante manipule o comportamento de um Large Language Model através de entradas maliciosas, fazendo com que o modelo ignore suas instruções originais e execute comandos não autorizados.

No contexto de um help desk com IA, o risco é amplificado porque:

1. O agente de IA possui **ferramentas integradas** (reset de senha, consulta ao AD, envio de email, etc.)
2. Qualquer usuário pode **abrir um ticket** — inclusive atacantes externos
3. A IA processa a entrada do usuário **sem sanitização adequada**
4. As ações executadas pelas ferramentas da IA são **difíceis de distinguir** de ações legítimas

---

## Vulnerabilidades Demonstradas

| ID | Vulnerabilidade | Categoria | Impacto |
|----|----------------|-----------|---------|
| ![LLM01](https://img.shields.io/badge/LLM01-Prompt_Injection-critical) | Prompt Injection | OWASP LLM Top 10 | 🔴 Crítico |
| ![LLM02](https://img.shields.io/badge/LLM02-Insecure_Output-red) | Insecure Output Handling | OWASP LLM Top 10 | 🔴 Alto |
| ![LLM06](https://img.shields.io/badge/LLM06-Info_Disclosure-orange) | Sensitive Information Disclosure | OWASP LLM Top 10 | 🟠 Alto |
| ![LLM08](https://img.shields.io/badge/LLM08-Excessive_Agency-critical) | Excessive Agency | OWASP LLM Top 10 | 🔴 Crítico |
| ![T1190](https://img.shields.io/badge/T1190-Public_App_Exploit-red) | Exploit Public-Facing Application | MITRE ATT&CK | 🔴 Crítico |
| ![T1078](https://img.shields.io/badge/T1078.002-Domain_Accounts-orange) | Valid Accounts: Domain Accounts | MITRE ATT&CK | 🟠 Alto |
| ![T1486](https://img.shields.io/badge/T1486-Ransomware-critical) | Data Encrypted for Impact | MITRE ATT&CK | 🔴 Crítico |

---

## OWASP LLM Top 10 — Mapeamento Completo

![OWASP LLM Top 10](assets/owasp-llm-top10.svg)

---

## Kill Chain do Ataque

![Kill Chain](assets/kill-chain.svg)

---

## Público-Alvo

- **CTOs e Diretores de TI** que estão avaliando ou já implementaram IA no help desk
- **Profissionais de segurança** que precisam compreender as novas superfícies de ataque criadas por sistemas de IA
- **Desenvolvedores** de sistemas de help desk com IA
- **Gestores de risco** que precisam quantificar o risco de IA em sistemas críticos

---

## Como Usar Este Material

### Para Palestrantes / Instrutores

1. Leia o `01-storytelling/historia.md` para entender a narrativa
2. Execute as fases de ataque em ordem (pasta `04-ataque/`)
3. Use o material de MITRE e OWASP para contextualização técnica
4. Encerre com as ações preventivas e proposta de valor

### Para Times de Segurança (Blue Team)

Use este material para:
- Desenvolver regras de detecção no SIEM para ataques a sistemas de IA
- Criar checklists de segurança para avaliação de sistemas de IA existentes
- Treinar o time em resposta a incidentes envolvendo IA

### Para Desenvolvedores

Foque em:
- `08-prevencao/acoes-preventivas.md` — código Python de sanitização e guardrails
- `04-ataque/fase-02-acesso-inicial.md` — como funciona o prompt injection
- `04-ataque/fase-03-comprometimento-ia.md` — quais ferramentas são mais perigosas

---

## Disclaimer Legal e Ético

> **AVISO IMPORTANTE**: Todo o conteúdo deste repositório é estritamente educacional. As técnicas, ferramentas e procedimentos aqui descritos devem ser utilizados APENAS em ambientes controlados, isolados e com autorização explícita. A execução destas técnicas em sistemas reais sem autorização constitui crime previsto na Lei 12.737/2012 (Lei Carolina Dieckmann) e pode resultar em penalidades criminais e civis.
>
> Este material foi desenvolvido para conscientização sobre segurança em sistemas de IA. O objetivo é que profissionais de segurança, desenvolvedores e gestores compreendam os riscos e implementem as devidas proteções.

---

## Autor

**Almir Meira Alves**
Diretor — Meira e Sousa Ltda | Educa com Talento

- 20+ anos de experiência em segurança da informação
- Especialista em Red Team, SOC, DevSecOps e IA aplicada à segurança
- LinkedIn: linkedin.com/in/almirmeira

---

## Evento

**Milvus Summit 2026**
Data: 19 de agosto de 2026
Local: São Paulo, SP
Palestra: "Operação Phantom Helper — Quando a IA do Help Desk Vira Arma"
Duração: 50 minutos + 10 minutos de Q&A

---

## Licença

Este material é de uso educacional. Redistribuição permitida com atribuição ao autor.
Proibido uso comercial sem autorização prévia.

---

*Desenvolvido em março de 2026 | Versão 1.0*
