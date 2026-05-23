# SDR Imobiliário Automatizado — WhatsApp + Minha Casa Minha Vida

Sistema de qualificação automatizada de leads imobiliários via WhatsApp, construído com arquitetura de State Machine + Classificador de Intenção. Opera em produção atendendo leads do programa Minha Casa Minha Vida na Bahia.

> O sistema atua como camada intermediária inteligente entre o lead e o corretor humano — qualificando, contextualizando e escalando apenas quando necessário.

---

## O Problema

Corretores imobiliários perdem tempo operacional respondendo manualmente centenas de leads vindos de Meta Ads, WhatsApp e Instagram. A maioria das perguntas é repetitiva. A maioria dos leads não está qualificada. O corretor não tem visibilidade do estágio de cada contato.

Um chatbot genérico resolve parte do problema, mas cria outro: respostas sem contexto, sem qualificação real, e sem controle sobre o que a IA diz sobre produtos imobiliários.

---

## A Decisão de Arquitetura Central

**O sistema não é um chatbot livre.**

A maioria das soluções similares usa um LLM para controlar o fluxo inteiro — o que resulta em respostas imprevisíveis, custos elevados e risco de a IA inventar informações sobre imóveis.

A decisão aqui foi diferente:

```
STATE MACHINE + CLASSIFICADOR + ENTITY EXTRACTION + TEMPLATES
```

A IA faz apenas o que ela faz bem: classificar intenção e extrair entidades. Toda lógica de negócio permanece determinística. As respostas são templates versionados no banco de dados, não geração livre.

**Por que isso importa:**
- Previsibilidade total sobre o que o sistema vai responder
- Custo de IA mínimo (uma chamada por mensagem, com enum fechado)
- Nenhum risco de alucinação sobre dados imobiliários
- Auditoria completa de cada decisão tomada

---

## Arquitetura: 4 Pilares em Série

```
[WhatsApp / Z-API]
        │
        ▼
┌──────────────────┐
│    PILAR 1       │  Entrada, Validação e Contexto
│                  │  • Validação do webhook
│                  │  • Transcrição de áudio (Whisper)
│                  │  • Debounce/Buffer (15s, via Supabase)
│                  │  • Normalização de telefone e mensagem
│                  │  • Busca ou criação do lead
└────────┬─────────┘
         │ contexto_inicial
         ▼
┌──────────────────┐
│    PILAR 2       │  Context Classification
│                  │  • Carrega histórico (últimas 10 msgs)
│                  │  • Carrega empreendimentos ativos do banco
│                  │  • 1 chamada de IA → intent + entities + confidence
│                  │  • Validação de enum em código (não na IA)
│                  │  • Fallback seguro se confidence < 0.6
└────────┬─────────┘
         │ contexto_classificado
         ▼
┌──────────────────┐
│    PILAR 3       │  Qualification Engine
│                  │  • Score de qualificação (0–100)
│                  │  • Match de empreendimento
│                  │  • Progressão de etapa (sem regressão)
│                  │  • Atualização do lead no Supabase
└────────┬─────────┘
         │ contexto_qualificado
         ▼
┌──────────────────┐
│    PILAR 4       │  Response Engine
│                  │  • Busca template por etapa + intent
│                  │  • Fallback para GPT se template não existe
│                  │  • Persistência (messages + flow_debug)
│                  │  • Envio via Z-API
│                  │  • Escalonamento para corretor
└──────────────────┘
         │
         ▼
[Lead recebe resposta no WhatsApp]
```

---

## Decisões Técnicas Relevantes

### Intent detection + Entity extraction em uma chamada única
O Pilar 2 faz uma única chamada ao GPT-4o-mini que retorna simultaneamente a intenção classificada, as entidades extraídas e o score de confiança. Isso evita custo duplo, reduz latência e elimina inconsistência entre classificação e extração feitas separadamente.

### Enum fechado com validação em código
A IA recebe um enum de 10 intenções possíveis e é instruída a nunca sair dele. Mesmo assim, o sistema valida o retorno em código antes de usar — a IA pode errar o formato, o código não pode confiar apenas nela.

### Proteção de regressão de etapa
O funil tem 7 etapas ordenadas. Uma vez que o lead avança, ele nunca regride — mesmo que o Pilar 2 classifique uma intenção que sugira etapa anterior. A lógica de progressão é determinística e protegida no Pilar 3.

### Score de qualificação com penalizações
```
renda informada:          +25
renda apta ao produto:    +20
empreendimento citado:    +20
nome real coletado:       +10
tipo de trabalho:         +10
intent positivo:          +15
intent objeção:           -15
intent desinteresse:      -30
tentativas excessivas:    -20
```

### Debounce via Supabase
WhatsApp frequentemente envia múltiplas mensagens em sequência. Uma janela de 15 segundos com lógica de "winner" no Supabase agrupa as mensagens antes de disparar o pipeline — evitando execuções paralelas e respostas duplicadas.

### Empreendimentos carregados dinamicamente
A lista de 28 empreendimentos (com aliases para menções informais) é carregada do Supabase a cada execução. Nunca está hardcoded no prompt. Isso permite adicionar ou desativar empreendimentos em produção sem tocar no fluxo.

### GPT fine-tuned no Pilar 4
O modelo de geração de resposta foi fine-tuned com 958 pares de conversas reais do corretor. O objetivo é manter o tom e estilo humano — o lead não pode perceber que está interagindo com um sistema automatizado.

---

## Funil de Etapas

```
primeiro_contato
      │
      ▼
descoberta_empreendimento
      │
      ▼
qualificacao
      │
      ▼
renda
      │
      ▼
documentacao
      │
      ▼
corretor ──► escalonamento humano
      │
      ▼
finalizado
```

Status transversais: `aguardando_resposta` | `lead_quente` | `lead_frio` | `reengajamento` | `pausado`

---

## Stack

| Camada | Tecnologia |
|---|---|
| Orquestração | n8n (self-hosted) |
| Banco de dados | Supabase (PostgreSQL) |
| IA — Classificação | GPT-4o-mini (temperature=0, json_object) |
| IA — Resposta | GPT-4o-mini fine-tuned (958 pares reais) |
| IA — Áudio | OpenAI Whisper |
| Gateway WhatsApp | Z-API |
| Origem de leads | Meta Ads, Instagram, orgânico |

---

## Schema Principal

### `leads`
Fonte da verdade do lead. Contém etapa, status, renda, tipo de trabalho, empreendimento de interesse, corretor responsável e histórico de tentativas.

### `messages`
Histórico completo de todas as mensagens (user e assistant), com `intent_detectada`, `etapa_momento` e `template_chave` registrados por mensagem.

### `response_templates`
Templates de resposta indexados por `etapa + intent`. Suportam variáveis como `{{nome}}`, `{{empreendimento}}`, `{{renda_min}}`. São a primeira opção de resposta — GPT é fallback.

### `flow_debug`
Registro de cada execução: qual intent foi detectada, qual template foi usado (ou se foi GPT), tempo de resposta, erros. Essencial para auditoria e melhoria contínua.

### `empreendimentos`
28 empreendimentos ativos com faixa MCMV, renda mínima, bairro, tipologia e aliases para reconhecimento informal.

---

## Variáveis de Ambiente

```env
OPENAI_API_KEY=
ZAPI_INSTANCE=
ZAPI_TOKEN=
ZAPI_CLIENT_TOKEN=
CORRETOR_TELEFONE=
SUPABASE_URL=
SUPABASE_SERVICE_KEY=
```

---

## Contexto de Negócio

O sistema opera com empreendimentos do programa **Minha Casa Minha Vida** na região de Salvador/BA e Camaçari, cobrindo as faixas 1, 2 e 3 (até R$ 9.600 de renda familiar, conforme Portaria MCID nº 333, abril/2026).

Os leads chegam principalmente via Meta Ads e são atendidos 24h, qualificados automaticamente e escalados para o corretor humano apenas quando atingem score suficiente ou solicitam atendimento direto.

---

## Débitos Técnicos Documentados

- Separador de mensagens no debounce usa `join('')` sem separador — pode prejudicar extração de entidades em mensagens concatenadas; trocar para `' / '`
- Aliases no Pilar 2 cobrem menções formais mas podem falhar em menções muito parciais (ex: "bela vista" sem "MRV") — melhorar dicionário de aliases
- Campo `programa` (mcmv | sbpe) não existe ainda na tabela `empreendimentos` — necessário quando houver produtos SBPE no portfólio

---

## Autor

Desenvolvido como projeto autoral de automação comercial aplicada ao mercado imobiliário da Bahia.
