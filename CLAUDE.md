# CLAUDE.md — Polymarket Trade Alert Bot

## O que é este projeto

Bot de alertas de trade em n8n que monitora breaking news no Twitter/X, usa Google Gemini 2.5 Flash para avaliar relevância e gerar queries de busca, pesquisa mercados na Polymarket e envia alertas via Telegram.

O projeto consiste em **um único arquivo**: `polymarket_trade_alert.json` — o workflow exportado do n8n.

---

## Regras de desenvolvimento

- **Toda modificação no workflow, bug fix ou nova feature deve ser feita em uma branch dedicada** (`feat/`, `fix/`, `refactor/`, `docs/`) e mergeada em `main` ao final.
- **Toda alteração que mude comportamento, adicione nodes ou corrija bugs deve ser acompanhada de atualização da documentação** em `docs/pages/` e neste `CLAUDE.md` (tabela de nodes, fluxo, propagação de NO_TRADE, etc.).
- Commits em inglês seguindo Conventional Commits — veja a seção [Git workflow](#git-workflow).

---

## Arquivo principal

**`polymarket_trade_alert.json`** — workflow n8n com todos os nodes, conexões e parâmetros inline (credenciais hardcoded).

Para editar o workflow, edite este JSON diretamente. A estrutura é:

```json
{
  "name": "Polymarket Trade Alert Bot",
  "nodes": [ { "id": "n1", ... }, { "id": "n2", ... } ],
  "connections": { ... },
  "settings": { ... }
}
```

---

## Arquitetura do pipeline

```
Schedule (30min) → Buscar Tweets (twitterapi.io) → Extrair Tweets ──┐
                → Buscar Reddit (worldnews+Economics) → Extrair Posts Reddit ─┘
                                                        → Merge Fontes
  → SplitInBatches (1 por vez) → Analisar Noticia (filtro 35min)
  → É Recente? → Preparar Prompt IA → Agente de IA (Gemini)
  → Extrair Query IA → É Relevante? → Buscar Mercados (Polymarket)
  → Filtrar Mercados → Preparar Validação IA → Validar Mercado IA
  → Extrair Validação → Preparar Sentimento IA → Agente Sentimento IA
  → Extrair Sentimento → Gerar Sinal → Verificar Oportunidade
  → Telegram Alert

Telegram Comando (mensagem do usuário) → Processar Comando → Responder Telegram
```

---

## IDs dos nodes

| ID | Nome | Função |
|---|---|---|
| n1 | Schedule Trigger | Dispara a cada 30 minutos |
| n2 | Buscar Tweets | twitterapi.io advanced search |
| n3 | Extrair Tweets | Normaliza array de tweets (inclui `author_verified` via `isBlueVerified`/`isVerified`) |
| n4 | Processar um por um | SplitInBatches(1) — loop por tweet |
| n5 | Analisar Noticia | Descarta tweets > 35 min |
| n6 | É Recente? | IF signal ≠ NO_TRADE |
| n7 | Buscar Mercados | GET /public-search?q= na Gamma API |
| n8 | Filtrar Mercados | Filtra mercados por keywords |
| n9 | Gerar Sinal | Monta mensagem HTML do Telegram |
| n10 | Verificar Oportunidade | IF signal === TRADE |
| n11 | Telegram Alert | Envia alerta com parse_mode HTML |
| n12 | Sem Oportunidade | noOp — encerra branch |
| n13 | Preparar Prompt IA | Sanitiza tweet, computa credibility_score, aplica pré-filtros geopolítico (< 1k seguidores) e YouTube (< 5k), monta prompt |
| n14 | Agente de IA | Basic LLM Chain (chainLlm) |
| n15 | Extrair Query IA | Parseia resposta do LLM |
| n16 | LLM Chat Model | Sub-nó do n14 via ai_languageModel |
| n17 | É Relevante? | IF signal ≠ NO_TRADE |
| n18 | Preparar Validação IA | Sanitiza tweet e monta prompt YES/NO; skip se NO_TRADE |
| n19 | Validar Mercado IA | Basic LLM Chain — confirma se mercado bate com notícia |
| n20 | LLM Chat Model - Validação | Sub-nó do n19 via ai_languageModel |
| n21 | Extrair Validação | Parseia YES/NO; propaga NO_TRADE se rejeitado ou skip_validation |
| n22 | Preparar Sentimento IA | Sanitiza tweet e monta prompt POSITIVE/NEGATIVE; skip se NO_TRADE |
| n23 | Agente Sentimento IA | Basic LLM Chain — detecta se notícia é positiva ou negativa para o YES |
| n24 | LLM Chat Model - Sentimento | Sub-nó do n23 via ai_languageModel |
| n25 | Extrair Sentimento | Parseia POSITIVE/NEGATIVE; propaga NO_TRADE se skip_sentiment |
| n29 | Buscar Reddit | GET reddit.com/r/worldnews+Economics/new.json — 25 posts mais recentes |
| n30 | Extrair Posts Reddit | Normaliza posts do Reddit para o mesmo formato de tweet (id, text, author, createdAt, url, source) |
| n31 | Merge Fontes | Combina tweets (n3) e posts Reddit (n30) em modo append antes do SplitInBatches |
| n26 | Telegram Comando | Telegram Trigger — recebe comandos de configuração do usuário |
| n27 | Processar Comando | Parseia /filtros, /ativar, /desativar; atualiza userFilters no static data |
| n28 | Responder Telegram | Envia resposta HTML ao usuário via Telegram |

---

## Credenciais hardcoded no JSON

Não há arquivo `.env` ou sistema de credenciais externo. Tudo está inline no `polymarket_trade_alert.json`:

- **n2 Buscar Tweets** — header `X-API-Key` com a chave da twitterapi.io
- **n11 Telegram Alert** — `chatId` hardcoded
- **n16 Google Gemini** — credencial referenciada por ID interno do n8n (configurada na instância)

---

## APIs utilizadas

### Twitter/X — twitterapi.io
- `GET https://api.twitterapi.io/twitter/tweet/advanced_search`
- Header: `X-API-Key`
- Query: `(breaking OR "just in") (election OR ...) -is:retweet lang:en`

### Polymarket — Gamma API
- **CORRETO:** `GET https://gamma-api.polymarket.com/public-search?q=<query>`
- **NÃO USAR:** `/markets?search=` — esse parâmetro é ignorado pela API e sempre retorna os mesmos mercados populares
- Resposta: `{ events: [ { title, slug, markets: [...] } ], pagination }`
- Cada mercado dentro de `events[].markets[]` tem: `question`, `slug`, `volumeNum`, `outcomePrices`, `active`, `closed`

### Google Gemini — via n8n LangChain
- Node: `@n8n/n8n-nodes-langchain.chainLlm` (Basic LLM Chain, typeVersion 1.4)
- Modelo: `models/gemini-3-flash-preview`
- Credencial tipo: `googlePalmApi`
- Output: `response.text` (não `response.content` nem `response.output`)

### Telegram
- Node: `n8n-nodes-base.telegram`
- `parse_mode: HTML` — obrigatório, Markdown quebra com caracteres especiais
- `chatId` hardcoded no node

---

## Decisões arquiteturais importantes

### Por que Basic LLM Chain e não AI Agent?
O Gemini bloqueia conteúdo violento via safety filters quando o nó é Agent. O `chainLlm` (Basic LLM Chain) não passa pelo mesmo mecanismo de safety e funciona corretamente.

### Por que sanitizar o tweet antes de enviar ao LLM?
Tweets com emojis e URLs causavam bloqueios do safety filter do Gemini. O n13 remove URLs (`https?://\S+`), asteriscos e emojis (surrogate pairs) antes de montar o prompt.

### Por que o LLM retorna NO_TRADE e não apenas a query?
O LLM avalia a relevância. Se o tweet não tem mercado provável na Polymarket, ele retorna `NO_TRADE` ou `NO_` (Gemini às vezes trunca). O check é `startsWith("NO_")`.

### Por que threshold dinâmico de keywords no n8?
- Query com 1–2 palavras → 1 match basta (query curta já é específica)
- Query com 3+ palavras → 2 matches obrigatórios (evita falsos positivos)

### Por que HTML e não Markdown no Telegram?
Markdown no Telegram falha quando o texto do tweet contém `*`, `_`, `[`, `]` etc. HTML com `esc()` (escapa `&`, `<`, `>`) é mais robusto.

### Por que `$('Extrair Query IA').first().json` no n8?
O n8 recebe dados do n7 (Buscar Mercados), mas precisa dos dados do tweet (author, tweet_id, ai_query). Como o SplitInBatches quebra a cadeia, o backreference direto ao n15 é necessário.

---

## Propagação de NO_TRADE

O sinal `NO_TRADE` pode surgir em:
1. **n5** — tweet já processado (deduplicação via static data, TTL 2h) ou com mais de 35 minutos → n6 descarta
2. **n13** — dois pré-filtros hard-coded sem custo de LLM:
   - Notícia geopolítica (war, missile, military, election, etc.) + autor < 1.000 seguidores → `pre_no_trade: true`
   - YouTube + autor < 5.000 seguidores → `pre_no_trade: true`
   - Também computa `credibility_score` (0–70) baseado em seguidores + verificação (`isBlueVerified`/`isVerified`)
   - Filtro de categoria: classifica o tweet em `geopolitica`, `crypto`, `economia` ou `esportes` por regex; retorna NO_TRADE se a categoria estiver desativada em `staticData.userFilters`
3. **n14/n15** — LLM julga irrelevante → n17 descarta
4. **n8** — nenhum mercado encontrado ou passou no filtro → n18/n21 propagam
5. **n21** — LLM de validação rejeitou o mercado → n22 recebe e propaga via skip_sentiment
6. **n9/n10** — qualquer NO_TRADE chega ao n9, que propaga; n10 descarta

---

## Reddit como fonte de notícias

### API utilizada
- **URL:** `GET https://www.reddit.com/r/worldnews+Economics/new.json?limit=25&sort=new`
- **Auth:** nenhuma (API pública)
- **Header obrigatório:** `User-Agent: n8n:polymarket-alert-bot:1.0`
- O operador `+` na URL busca os dois subreddits em uma única chamada

### Normalização (n30)
Posts do Reddit são convertidos para o mesmo formato de tweet que o restante do pipeline espera:

| Campo Reddit | Campo normalizado | Observação |
|---|---|---|
| `data.id` | `id` | Prefixado com `reddit_` para evitar colisão com IDs de tweet |
| `data.title` + `data.selftext` | `text` | selftext limitado a 200 chars; ignorado se `[removed]` |
| `data.author` | `author` | username do Reddit |
| `max(score * 100, 10000)` | `author_followers` | Proxy: score alto → mais credibilidade; mínimo 10k para não ser barrado pelo filtro geopolítico |
| `data.created_utc * 1000` | `createdAt` | Unix → ISO string |
| `https://reddit.com` + `permalink` | `url` | Link direto ao post |
| `"reddit"` | `source` | Distingue de tweets no n9 |
| `data.stickied === true` | filtrado | Posts fixados são descartados |

### Mensagem Telegram para posts Reddit
Quando `source === "reddit"`, o n9 usa formato diferente:
```
📰 Reddit r/worldnews — u/autor:
Título do post...
```
Em vez de:
```
🐦 Tweet de @autor:
```
O link de origem usa 📎 em vez de 🐦.

---

## Filtros de categoria por usuário

O usuário pode configurar via Telegram quais categorias de mercado deseja monitorar. O estado é persistido em `$getWorkflowStaticData('global').userFilters` entre execuções.

### Categorias e keywords de detecção

| Categoria | Keywords principais |
|---|---|
| `geopolitica` | war, missile, military, invasion, election, nato, ceasefire, coup, assassination |
| `crypto` | bitcoin, ethereum, blockchain, defi, nft, binance, coinbase, solana, xrp |
| `economia` | fed, rate cut/hike, inflation, gdp, tariff, trade deal, recession, nasdaq |
| `esportes` | nba, nfl, world cup, olympics, superbowl, fifa, formula 1, ufc |

Tweets que não casam com nenhuma categoria passam livremente (sem classificação = não filtrado).

### Comandos Telegram

| Comando | Exemplo | Efeito |
|---|---|---|
| `/filtros` | `/filtros` | Exibe status de cada categoria |
| `/ativar <cat>` | `/ativar crypto` | Ativa a categoria |
| `/desativar <cat>` | `/desativar esportes` | Desativa a categoria |
| `/help` | `/help` | Lista todos os comandos |

O argumento aceita acentos ou não (`geopolítica` = `geopolitica`). Não é possível desativar a última categoria ativa.

### Static data

```json
{
  "userFilters": {
    "geopolitica": true,
    "crypto": true,
    "economia": true,
    "esportes": true
  }
}
```

Default: todas ativas (comportamento idêntico ao estado anterior à feature).

---

## Formato da mensagem Telegram

```
🚨 ALERTA DE TRADE - POLYMARKET

🐦 Tweet de @autor:
Texto do tweet (até 120 chars)...

📊 Mercado:
Pergunta do mercado

🎯 Direção: BUY_YES
💪 Confiança: 75%
📈 Movimento: 28% → 48%
💰 Volume: $970,440
🤖 Query IA: iran nuclear deal

🐦 https://x.com/autor/status/ID
🔗 https://polymarket.com/event/slug
```

---

## Git workflow

### Branches
Every improvement, bug fix, or new feature must be developed in a dedicated branch:

```bash
git checkout -b fix/short-description      # bug fixes
git checkout -b feat/short-description     # new features
git checkout -b docs/short-description     # documentation only
git checkout -b refactor/short-description # refactors
```

After implementing, commit and open a PR (or merge directly into `main` if working solo):

```bash
git add <files>
git commit -m "type: short description"
git push -u origin <branch>
# then merge into main
```

### Commit message style
All commit messages must be in **English**, following Conventional Commits:

```
type: short imperative description (max 72 chars)

Optional body explaining the why, not the what.
```

**Types:**

| Type | When to use |
|---|---|
| `feat` | New feature or capability |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `refactor` | Code restructure with no behavior change |
| `chore` | Maintenance (deps, config, tooling) |

**Examples:**
```
feat: add LLM market validation before sending alert
fix: switch search endpoint to /public-search for real text search
docs: add CLAUDE.md with project context for Claude Code
refactor: extract tweet sanitizer into reusable function
```

**Rules:**
- Imperative mood: "add" not "added", "fix" not "fixed"
- No period at the end
- No uppercase first letter after the colon
- Reference the *why* in the body when the change isn't self-evident

---

## Documentação adicional

Toda a documentação está em `docs/pages/`:
- `architecture.md` — diagrama completo do fluxo
- `nodes.md` — referência técnica de cada node
- `signal-logic.md` — cálculo de direção e confiança
- `telegram.md` — template da mensagem
- `setup.md` — instruções de importação no n8n
- `api-reference.md` — referência das APIs externas
- `twitter.md` — detalhes da query de busca
