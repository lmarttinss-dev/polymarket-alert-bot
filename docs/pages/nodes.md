# Referência dos Nodes

Descrição técnica de cada node do workflow.

---

## n1 — Schedule Trigger

| Propriedade | Valor |
|---|---|
| Tipo | `n8n-nodes-base.scheduleTrigger` |
| typeVersion | 1.1 |
| Intervalo | A cada 30 minutos |

Dispara a execução automaticamente. Não produz dados.

---

## n2 — Buscar Tweets

| Propriedade | Valor |
|---|---|
| Tipo | `n8n-nodes-base.httpRequest` |
| typeVersion | 4.1 |
| Método | GET |
| URL | `https://api.twitterapi.io/twitter/tweet/advanced_search` |
| Timeout | 10.000ms |

**Header:**
```
X-API-Key: <hardcoded no JSON>
```

**Query params:**

| Parâmetro | Valor |
|---|---|
| `query` | `(breaking OR "just in") (election OR rates OR fed OR crypto OR bitcoin OR trump OR tariff OR ceasefire OR war) -is:retweet lang:en` |
| `queryType` | `Latest` |

**Saída:** objeto `{ tweets: [...], has_next_page, next_cursor }`

---

## n3 — Extrair Tweets

| Propriedade | Valor |
|---|---|
| Tipo | `n8n-nodes-base.code` |
| typeVersion | 2 |
| Mode | `runOnceForAllItems` |

Normaliza a resposta da twitterapi.io. Extrai o array `tweets` e transforma cada item em um objeto padronizado.

**Saída (por tweet):**
```json
{
  "id": "1234567890",
  "text": "BREAKING: Fed signals rate cut...",
  "author": "Reuters",
  "author_followers": 24000000,
  "createdAt": "Tue Apr 04 12:00:30 +0000 2026",
  "retweetCount": 45,
  "likeCount": 234,
  "viewCount": 5000,
  "url": "https://x.com/Reuters/status/1234567890",
  "isReply": false
}
```

**Saída (sem tweets):**
```json
{ "signal": "NO_TRADE", "reason": "Nenhum tweet retornado pela API" }
```

---

## n4 — Processar um por um

| Propriedade | Valor |
|---|---|
| Tipo | `n8n-nodes-base.splitInBatches` |
| typeVersion | 1 |
| Batch Size | 1 |

Processa cada tweet individualmente através do pipeline. Outputs:
- **Output 0** → próximo item → `Analisar Noticia`
- **Output 1** → todos processados → fim da execução

Recebe de volta (`Telegram Alert` e `Sem Oportunidade`) para avançar ao próximo tweet.

---

## n5 — Analisar Noticia

| Propriedade | Valor |
|---|---|
| Tipo | `n8n-nodes-base.code` |
| typeVersion | 2 |
| Mode | `runOnceForAllItems` |

**Lógica:**
1. Propaga `NO_TRADE` se vier do n3
2. Calcula idade do tweet via `createdAt` — descarta se > 35 min

> A extração de keywords foi removida. A análise de relevância é feita pelo LLM (n14).

**Saída:**
```json
{
  "news_text": "BREAKING: Fed signals rate cut in September...",
  "tweet_id": "1234567890",
  "tweet_url": "https://x.com/Reuters/status/1234567890",
  "author": "Reuters",
  "author_followers": 24000000,
  "timestamp": "2026-04-04T12:01:00.000Z"
}
```

---

## n6 — É Recente?

| Propriedade | Valor |
|---|---|
| Tipo | `n8n-nodes-base.if` |
| typeVersion | 2 |
| Condição | `$json.signal` ≠ `"NO_TRADE"` |

- **Output 0 (true)** → `Preparar Prompt IA`
- **Output 1 (false)** → `Sem Oportunidade`

---

## n13 — Preparar Prompt IA

| Propriedade | Valor |
|---|---|
| Tipo | `n8n-nodes-base.code` |
| typeVersion | 2 |
| Mode | `runOnceForAllItems` |

Sanitiza o texto do tweet e monta o prompt para o LLM. Também aplica um **pré-filtro hard-coded** que descarta fontes de baixa credibilidade antes de chamar o Gemini.

**Pré-filtro (sem custo de LLM):**

Se o tweet contém uma referência ao YouTube (`via @YouTube`, `youtube.com` ou `youtu.be`) **E** o autor tem menos de 5.000 seguidores, o node retorna imediatamente com `pre_no_trade: true`, sem montar prompt nem consumir tokens.

```js
const hasYouTubeRef = /via @YouTube|youtube\.com|youtu\.be/i.test(originalText);
const isLowCredibility = followers < 5000;

if (hasYouTubeRef && isLowCredibility) {
  return [{ json: { ...data, ai_prompt: "NO_TRADE", pre_no_trade: true,
    pre_no_trade_reason: "YouTube link from low-follower account (< 5000 followers)" } }];
}
```

**Sanitização aplicada (quando passa do pré-filtro):**
- Remove URLs (`https://...`)
- Remove asteriscos (`*`) — evita filtros de segurança do Gemini
- Remove emojis (surrogate pairs Unicode)
- Normaliza espaços

**Prompt enviado ao LLM:**
```
You are a Polymarket prediction market expert. Analyze this tweet...

Tweet: "<texto sanitizado>"
Author: @<autor> (<seguidores> followers)

TASK: If relevant → respond with 2-5 word search query
      If not relevant → respond with exactly: NO_TRADE

DISQUALIFY with NO_TRADE if ANY of the following apply:
- Tweet shares/links to a YouTube video ("via @YouTube", "youtu.be", etc.)
- Author has fewer than 10,000 followers AND claim is unverified
- Tweet text is clearly a video title or clickbait headline
- Reply, personal opinion, joke, insult, or spam
```

**Saída (pré-filtro ativado):** `{ ...dados_do_tweet, ai_prompt: "NO_TRADE", pre_no_trade: true, pre_no_trade_reason: "..." }`

**Saída (fluxo normal):** `{ ...dados_do_tweet, ai_prompt: "..." }`

---

## n14 — Agente de IA

| Propriedade | Valor |
|---|---|
| Tipo | `@n8n/n8n-nodes-langchain.chainLlm` |
| typeVersion | 1.4 |
| Prompt | `={{ $json.ai_prompt }}` |

Basic LLM Chain conectado ao sub-nó `LLM Chat Model` via `ai_languageModel`.

**Saída:** `{ text: "israel iran war" }` ou `{ text: "NO_TRADE" }`

---

## n16 — LLM Chat Model

| Propriedade | Valor |
|---|---|
| Tipo | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` |
| typeVersion | 1 |
| Modelo | `models/gemini-2.5-flash` |
| Max Output Tokens | 50 |

Sub-nó conectado ao `Agente de IA` via `ai_languageModel`. Não aparece no canvas principal.

**Credencial:** `Google Gemini API` (tipo: Google Gemini (PaLm) API)

---

## n15 — Extrair Query IA

| Propriedade | Valor |
|---|---|
| Tipo | `n8n-nodes-base.code` |
| typeVersion | 2 |
| Mode | `runOnceForAllItems` |

Parseia a saída do LLM. Tenta os campos `text`, `response` e `output` em sequência.

**Lógica:**
1. Se `newsData.pre_no_trade === true` (flag setada pelo pré-filtro do n13) → retorna `{ signal: "NO_TRADE" }` imediatamente, ignorando a resposta do LLM
2. Se output vazio ou começar com `"NO_"` → retorna `{ signal: "NO_TRADE" }`
3. Caso contrário → retorna dados do tweet + `search_query` + `ai_query`

**Saída (relevante):**
```json
{
  "news_text": "...",
  "tweet_id": "...",
  "author": "Reuters",
  "author_followers": 24000000,
  "search_query": "fed rate cut",
  "ai_query": "fed rate cut",
  "timestamp": "..."
}
```

---

## n17 — É Relevante?

| Propriedade | Valor |
|---|---|
| Tipo | `n8n-nodes-base.if` |
| typeVersion | 2 |
| Condição | `$json.signal` ≠ `"NO_TRADE"` |

- **Output 0 (true)** → `Buscar Mercados`
- **Output 1 (false)** → `Sem Oportunidade`

---

## n7 — Buscar Mercados

| Propriedade | Valor |
|---|---|
| Tipo | `n8n-nodes-base.httpRequest` |
| typeVersion | 4.1 |
| URL | `https://gamma-api.polymarket.com/public-search` |
| Timeout | 10.000ms |

Parâmetros: `q={{ $json.search_query }}`

> **Por que `/public-search` e não `/markets`?** O endpoint `/markets?search=` ignora completamente o parâmetro de busca e retorna sempre os mesmos mercados populares por ordem de criação. O `/public-search?q=` é o endpoint de busca por texto real usado pelo site da Polymarket — retorna eventos relevantes com seus mercados aninhados.

**Saída:** `{ events: [...], pagination: { hasMore, totalResults } }`

---

## n8 — Filtrar Mercados

| Propriedade | Valor |
|---|---|
| Tipo | `n8n-nodes-base.code` |
| typeVersion | 2 |
| Mode | `runOnceForAllItems` |

**Processamento da resposta do `/public-search`:**
- A resposta tem estrutura `{ events: [...] }` onde cada evento contém `markets: [...]`
- Extrai todos os mercados de todos os eventos, adicionando `eventSlug` e `eventTitle` a cada mercado

**Filtros aplicados (em ordem):**
1. Exclui mercados com `active === false`, `closed === true` ou `archived === true`
2. `volume > $500`
3. **Match de relevância:** palavras da `ai_query` devem aparecer em pelo menos um dos campos: título da pergunta, primeiros 300 chars da descrição, `eventTitle` ou `groupItemTitle`
   - Query com 1–2 palavras → **1 match** basta
   - Query com 3+ palavras → **2 matches** obrigatórios
4. Ordena por volume decrescente
5. Retorna top 5

Usa backreference `$('Extrair Query IA').first().json` para recuperar dados do tweet.

---

## n9 — Gerar Sinal

| Propriedade | Valor |
|---|---|
| Tipo | `n8n-nodes-base.code` |
| typeVersion | 2 |
| Mode | `runOnceForAllItems` |

Calcula direção, confiança e monta mensagem Telegram em HTML. Inclui bônus de +5% de confiança para autores com > 100k seguidores.

Aplica `esc()` em todos os campos dinâmicos (tweet, título do mercado, autor) para escapar `&`, `<`, `>` antes de inserir no HTML.

Ver [Lógica de Sinal](./signal-logic.md) para detalhes.

**Saída:**
```json
{
  "signal": "TRADE",
  "direction": "BUY_YES",
  "confidence": 83,
  "expected_from": 22,
  "expected_to": 42,
  "market_question": "Will the Fed cut rates in September 2026?",
  "market_url": "https://polymarket.com/event/...",
  "author": "Reuters",
  "author_followers": 24000000,
  "ai_query": "fed rate cut",
  "telegram_message": "<b>ALERTA DE TRADE...</b>",
  "timestamp": "2026-04-04T12:01:00.000Z"
}
```

---

## n10 — Verificar Oportunidade

| Propriedade | Valor |
|---|---|
| Tipo | `n8n-nodes-base.if` |
| typeVersion | 2 |
| Condição | `$json.signal` equals `"TRADE"` |

- **Output 0 (true)** → `Telegram Alert`
- **Output 1 (false)** → `Sem Oportunidade`

---

## n11 — Telegram Alert

| Propriedade | Valor |
|---|---|
| Tipo | `n8n-nodes-base.telegram` |
| typeVersion | 1.1 |
| Chat ID | hardcoded no JSON |
| Parse Mode | **HTML** |

**Credencial:** `Telegram Bot API`

Após envio, retorna ao `Processar um por um` para o próximo tweet.

---

## n12 — Sem Oportunidade

| Propriedade | Valor |
|---|---|
| Tipo | `n8n-nodes-base.noOp` |
| typeVersion | 1 |

Encerra silenciosamente branches `NO_TRADE` e retorna ao loop para processar o próximo tweet. Recebe de três origens: `É Recente?` (false), `É Relevante?` (false) e `Verificar Oportunidade` (false).

---

## n18 — Preparar Validação IA

| Propriedade | Valor |
|---|---|
| Tipo | `n8n-nodes-base.code` |
| typeVersion | 2 |
| Mode | `runOnceForAllItems` |

Prepara o prompt de validação para o LLM.

**Sanitização do tweet:** aplica o mesmo `sanitize()` do n13 (remove URLs, asteriscos e emojis) antes de montar o prompt, evitando bloqueios do safety filter do LLM que causavam `text: ""`.

**Skip sem custo:** se o input já for `NO_TRADE` ou sem mercados, seta `skip_validation: true` e `ai_validation_prompt: 'YES'` — o chainLlm n19 ainda executa, mas o n21 detecta o flag e ignora a resposta, sem desperdiçar tokens em uma chamada desnecessária.

**Prompt gerado (quando há mercado):**
> "Does this market DIRECTLY relate to the news in the tweet? Answer YES if the news could move the market probability. Answer NO if the connection is weak, indirect, or about a different topic."

---

## n19 — Validar Mercado IA

| Propriedade | Valor |
|---|---|
| Tipo | `@n8n/n8n-nodes-langchain.chainLlm` |
| typeVersion | 1.4 |

Basic LLM Chain que chama o Gemini com o prompt de validação. Sub-nó: `LLM Chat Model - Validação` (n20).

---

## n20 — LLM Chat Model - Validação

| Propriedade | Valor |
|---|---|
| Tipo | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` |
| Modelo | `models/gemini-2.5-flash` |
| maxOutputTokens | 10 |

Sub-nó do `Validar Mercado IA`. maxOutputTokens reduzido a 10 pois a resposta esperada é apenas `YES` ou `NO`.

---

## n21 — Extrair Validação

| Propriedade | Valor |
|---|---|
| Tipo | `n8n-nodes-base.code` |
| typeVersion | 2 |
| Mode | `runOnceForAllItems` |

Parseia a resposta do LLM de validação via backreference `$('Filtrar Mercados').first().json` (o chainLlm não propaga o contexto anterior).

**Lógica de decisão:**
1. Se `originalData.signal === 'NO_TRADE'` ou `skip_validation === true` → propaga `originalData` direto, ignorando a resposta do LLM
2. Se resposta começa com `YES` → passa `originalData` (com `markets[]`) para `Preparar Sentimento IA`
3. Qualquer outra resposta (incluindo `NO`, vazio ou resposta inesperada) → retorna `{ signal: "NO_TRADE", reason: "Validação LLM: mercado não relevante para esta notícia" }`

---

## n22 — Preparar Sentimento IA

| Propriedade | Valor |
|---|---|
| Tipo | `n8n-nodes-base.code` |
| typeVersion | 2 |
| Mode | `runOnceForAllItems` |

Monta o prompt de análise de sentimento para o LLM.

**Skip sem custo:** se o input já for `NO_TRADE` ou sem mercados, seta `skip_sentiment: true` com prompt vazio — o chainLlm n23 ainda executa, mas o n25 detecta o flag e ignora a resposta.

**Sanitização:** aplica o mesmo `sanitize()` do n13 (remove URLs, asteriscos e emojis) antes de montar o prompt.

**Prompt gerado (quando há mercado):**
> "Is this news POSITIVE or NEGATIVE for the YES outcome of this market? POSITIVE: the news increases the probability that YES is correct. NEGATIVE: the news decreases the probability that YES is correct."

---

## n23 — Agente Sentimento IA

| Propriedade | Valor |
|---|---|
| Tipo | `@n8n/n8n-nodes-langchain.chainLlm` |
| typeVersion | 1.4 |
| Prompt | `={{ $json.ai_sentiment_prompt }}` |

Basic LLM Chain que chama o Gemini com o prompt de sentimento. Sub-nó: `LLM Chat Model - Sentimento` (n24).

---

## n24 — LLM Chat Model - Sentimento

| Propriedade | Valor |
|---|---|
| Tipo | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` |
| Modelo | `models/gemini-2.5-flash` |
| maxOutputTokens | 10 |

Sub-nó do `Agente Sentimento IA`. maxOutputTokens reduzido a 10 pois a resposta esperada é apenas `POSITIVE` ou `NEGATIVE`.

---

## n25 — Extrair Sentimento

| Propriedade | Valor |
|---|---|
| Tipo | `n8n-nodes-base.code` |
| typeVersion | 2 |
| Mode | `runOnceForAllItems` |

Parseia a resposta do LLM de sentimento via backreference `$('Preparar Sentimento IA').first().json`.

**Lógica de decisão:**
1. Se `skip_sentiment === true` ou `signal === 'NO_TRADE'` → propaga `originalData` direto, sem alterar
2. Se resposta começa com `NEGATIVE` → adiciona `{ sentiment: "NEGATIVE" }` ao objeto
3. Qualquer outra resposta → assume `{ sentiment: "POSITIVE" }` (default seguro)

**Saída:** `originalData` com campo adicional `sentiment: "POSITIVE" | "NEGATIVE"`
