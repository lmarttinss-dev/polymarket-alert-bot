# API Reference

Referência das APIs externas consumidas pelo workflow.

---

## twitterapi.io — Advanced Search

**Endpoint:**
```
GET https://api.twitterapi.io/twitter/tweet/advanced_search
```

**Autenticação:**
```
X-API-Key: <TWITTERAPI_IO_KEY>
```

**Parâmetros:**

| Parâmetro | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `query` | string | Sim | Expressão de busca no formato Twitter |
| `queryType` | string | Sim | `Latest` ou `Top` |
| `cursor` | string | Não | Paginação (primeira página: omitir) |

**Exemplo de requisição:**
```
GET https://api.twitterapi.io/twitter/tweet/advanced_search
    ?query=(breaking OR "just in") (fed OR rates) -is:retweet lang:en
    &queryType=Latest
X-API-Key: abc123...
```

**Resposta:**
```json
{
  "tweets": [
    {
      "id": "1234567890",
      "text": "BREAKING: Fed signals rate cut...",
      "url": "https://x.com/Reuters/status/1234567890",
      "createdAt": "Tue Apr 04 12:00:30 +0000 2026",
      "retweetCount": 45,
      "likeCount": 234,
      "viewCount": 5000,
      "isReply": false,
      "author": {
        "userName": "Reuters",
        "name": "Reuters",
        "followers": 24000000
      }
    }
  ],
  "has_next_page": true,
  "next_cursor": "cursor_token_here"
}
```

**Custo:** $0,15 por 1.000 tweets retornados.

**Documentação:** [docs.twitterapi.io](https://docs.twitterapi.io)

---

## Polymarket Gamma API — Markets

**Endpoint:**
```
GET https://gamma-api.polymarket.com/markets
```

**Autenticação:** nenhuma — API pública.

**Parâmetros:**

| Parâmetro | Valor | Descrição |
|---|---|---|
| `search` | `={{ $json.search_query }}` | Keywords do tweet |
| `active` | `true` | Somente mercados ativos |
| `closed` | `false` | Exclui encerrados |
| `limit` | `20` | Máximo de resultados |

**Campos utilizados da resposta:**

| Campo | Uso |
|---|---|
| `id` / `slug` | Geração da URL do mercado |
| `question` / `title` | Pergunta exibida no alerta |
| `outcomePrices` | Preço atual YES/NO |
| `volumeNum` / `volume` | Filtro e bônus de confiança |
| `active`, `closed`, `archived` | Filtros de status |

---

## Telegram Bot API — Send Message

**Endpoint (gerenciado pelo n8n):**
```
POST https://api.telegram.org/bot<TOKEN>/sendMessage
```

**Payload:**

| Campo | Valor |
|---|---|
| `chat_id` | `$env.TELEGRAM_CHAT_ID` |
| `text` | `$json.telegram_message` |
| `parse_mode` | `Markdown` |

**Documentação:** [core.telegram.org/bots/api#sendmessage](https://core.telegram.org/bots/api#sendmessage)
