# Arquitetura

## Fluxo Completo

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      POLYMARKET TRADE ALERT BOT                         │
└─────────────────────────────────────────────────────────────────────────┘

┌──────────────────┐
│ Schedule Trigger │  n1 — Dispara a cada 30 minutos automaticamente
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Buscar Tweets   │  n2 — twitterapi.io advanced search
└────────┬─────────┘  Query: breaking news sobre temas Polymarket
         │
         ▼
┌──────────────────┐
│ Extrair Tweets   │  n3 — Normaliza array de tweets
└────────┬─────────┘
         │
         ▼
┌──────────────────────┐
│ Processar um por um  │◄──────────────────────────────────────────────┐
│  SplitInBatches      │  batchSize: 1                                  │
└──────┬───────────────┘                                                │
       │ output 0 (item)          output 1 (done) → fim                │
       ▼                                                                │
┌──────────────────┐                                                    │
│ Analisar Noticia │  n5 — Filtra tweets com mais de 35 minutos        │
└──────┬───────────┘  Saída: { news_text, tweet_id, author, ... }      │
       │              Saída (tweet antigo): { signal: "NO_TRADE" }     │
       ▼                                                                │
┌──────────────────┐                                                    │
│   É Recente?     │  n6 — IF: signal ≠ NO_TRADE                       │
└──────┬───────────┘                                                    │
       │ true                    false ──────────────────────────────►│ │
       ▼                                                               │ │
┌──────────────────────┐                                               │ │
│  Preparar Prompt IA  │  n13 — Sanitiza tweet e monta prompt          │ │
└──────┬───────────────┘  Remove URLs, asteriscos, emojis              │ │
       │                                                               │ │
       ▼                                                               │ │
┌──────────────────┐                                                   │ │
│   Agente de IA   │  n14 — Basic LLM Chain (chainLlm)                │ │
│  (Gemini 2.5)    │  Decide se tweet é relevante e gera query        │ │
└──────┬───────────┘  Retorna: "israel iran war" ou "NO_TRADE"        │ │
       │                                                               │ │
       ▼                                                               │ │
┌──────────────────────┐                                               │ │
│  Extrair Query IA    │  n15 — Parseia resposta do LLM               │ │
└──────┬───────────────┘  NO_TRADE → { signal: "NO_TRADE" }           │ │
       │                                                               │ │
       ▼                                                               │ │
┌──────────────────┐                                                   │ │
│   É Relevante?   │  n17 — IF: signal ≠ NO_TRADE                     │ │
└──────┬───────────┘                                                   │ │
       │ true                    false ──────────────────────────────► │ │
       ▼                                                                │ │
┌──────────────────┐                                                    │ │
│ Buscar Mercados  │  n7 — GET gamma-api.polymarket.com/public-search   │ │
└──────┬───────────┘  q={{ search_query }} — busca por texto real      │ │
       │                                                                │ │
       ▼                                                                │ │
┌──────────────────┐                                                    │ │
│ Filtrar Mercados │  n8 — Filtra por volume > $500                    │ │
└──────┬───────────┘  Match ≥ 2 palavras da query no título/descrição  │ │
       │              Retorna top 5 por volume                          │ │
       ▼                                                                │ │
┌──────────────────┐                                                    │ │
│  Gerar Sinal     │  n9 — Calcula direção, confiança, mensagem HTML   │ │
└──────┬───────────┘                                                    │ │
       │                                                                │ │
       ▼                                                                │ │
┌────────────────────────┐                                              │ │
│ Verificar Oportunidade │  n10 — IF: signal === "TRADE"               │ │
└────────┬───────────────┘                                              │ │
         │                                                              │ │
    ┌────┴─────────┐                                                    │ │
   SIM            NÃO                                                   │ │
    │               │                                                   │ │
    ▼               ▼                                                   │ │
┌──────────┐  ┌──────────────┐                                          │ │
│ Telegram │  │     Sem      │◄──────────────────────────────────────── │ │
│  Alert   │  │ Oportunidade │                                           │ │
│  (n11)   │  │    (n12)     │◄────────────────────────────────────────┘ │
└────┬─────┘  └──────┬───────┘                                           │
     │               │                                                   │
     └───────┬───────┘                                                   │
             │                                                           │
             └───────────────────────────────────────────────────────► (loop)
```

---

## Sub-nó: LLM Chat Model

O node `LLM Chat Model` (n16) é um sub-nó conectado ao `Agente de IA` via conexão `ai_languageModel`. Não aparece no fluxo principal mas é requisito para o nó de LLM funcionar.

```
LLM Chat Model (n16)
  └── ai_languageModel ──► Agente de IA (n14)
```

---

## Padrão de Loop com SplitInBatches

O node `Processar um por um` (SplitInBatches) implementa o loop que processa cada tweet individualmente através de todo o pipeline.

```
Tweets recebidos: [T1, T2, T3, ..., T10]

Ciclo 1:  SplitInBatches → T1 → pipeline → resultado → SplitInBatches
Ciclo 2:  SplitInBatches → T2 → pipeline → resultado → SplitInBatches
...
Ciclo 10: SplitInBatches → T10 → pipeline → resultado → SplitInBatches
          SplitInBatches → output 1 (done) → fim da execução
```

**Por que processar 1 por vez e não em paralelo?**
- Cada tweet faz chamadas à API do Gemini e da Polymarket — serializar evita rate limiting
- Um tweet pode gerar alerta, o seguinte não — o loop garante processamento independente
- O contexto de cada tweet se mantém isolado durante o pipeline

---

## Propagação de NO_TRADE

O sinal `NO_TRADE` pode ser emitido em quatro pontos diferentes:

```
n5 Analisar Noticia ──► NO_TRADE (tweet > 35 min)
                              │
                              ▼
n6 É Recente? ────────► branch false → Sem Oportunidade → próximo tweet

n14 Agente de IA ─────► LLM retorna "NO_TRADE" (tweet irrelevante)
                              │
                              ▼
n15 Extrair Query IA ──► { signal: "NO_TRADE" }
                              │
                              ▼
n17 É Relevante? ──────► branch false → Sem Oportunidade → próximo tweet

n8 Filtrar Mercados ───► NO_TRADE (nenhum mercado com ≥ 2 palavras match)
                              │
                              ▼
n9 Gerar Sinal ────────► detecta NO_TRADE, propaga sem alterar
                              │
                              ▼
n10 Verificar Oportunidade → branch NÃO → Sem Oportunidade → próximo tweet
```

---

## Responsabilidades por Camada

| Camada | Node | Responsabilidade |
|---|---|---|
| Agendamento | Schedule Trigger | Disparar execução autonomamente |
| Coleta | Buscar Tweets | Buscar breaking news em tempo real |
| Normalização | Extrair Tweets | Padronizar estrutura dos tweets |
| Iteração | Processar um por um | Garantir processamento isolado por tweet |
| Filtragem temporal | Analisar Noticia | Descartar tweets com mais de 35 minutos |
| Roteamento | É Recente? | Bifurcar tweets recentes dos descartados |
| Preparação | Preparar Prompt IA | Sanitizar tweet e montar prompt para LLM |
| Inteligência | Agente de IA + Gemini | Avaliar relevância e gerar query de busca |
| Extração | Extrair Query IA | Parsear saída do LLM, detectar NO_TRADE |
| Roteamento | É Relevante? | Bifurcar tweets relevantes dos descartados pelo LLM |
| Dados externos | Buscar Mercados | Única integração com Polymarket |
| Filtragem | Filtrar Mercados | Garantir relevância (threshold dinâmico) e qualidade dos mercados |
| Decisão | Gerar Sinal | Calcular direção, confiança e montar mensagem |
| Roteamento | Verificar Oportunidade | Bifurcar entre alerta e descarte |
| Saída | Telegram Alert | Notificar oportunidade em HTML |
| Descarte | Sem Oportunidade | Encerrar silenciosamente branches sem oportunidade |

---

## Propagação de Dados entre Nodes

```
Schedule Trigger ──────────► (trigger vazio)
                                    │
Buscar Tweets ─────────────► { tweets: [...] }
                                    │
Extrair Tweets ─────────────► [ { id, text, author, author_followers,
                                   createdAt, url, isReply, ... } ]
                                    │
Processar um por um ────────► { id, text, author, ... }  (1 tweet)
                                    │
Analisar Noticia ───────────► { news_text, tweet_id, tweet_url,
                                 author, author_followers, timestamp }
                                    │
Preparar Prompt IA ─────────► { ...dados, ai_prompt: "..." }
                                    │
Agente de IA ───────────────► { text: "israel iran war" }
                                    │
Extrair Query IA ───────────► { news_text, tweet_id, tweet_url,
                                 author, author_followers,
                                 search_query, ai_query, timestamp }
                                    │
Buscar Mercados ────────────► { events: [ { title, markets: [...] } ], pagination }
                                    │
                 backreference ◄────┘
                 $('Extrair Query IA').first().json
                                    │
Filtrar Mercados ───────────► { markets[], market_count,
                                 news_text, tweet_id, author,
                                 author_followers, ai_query }
                                    │
Gerar Sinal ────────────────► { signal, direction, confidence,
                                 expected_from, expected_to,
                                 market_question, market_url,
                                 volume, tweet_id, ai_query,
                                 telegram_message, timestamp }
```
