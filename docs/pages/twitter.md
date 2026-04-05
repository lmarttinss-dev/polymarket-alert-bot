# Integração com Twitter/X via twitterapi.io

O workflow usa o serviço [twitterapi.io](https://twitterapi.io) como proxy para a API do Twitter/X, evitando a necessidade de aprovação e o custo do plano Basic oficial ($100/mês).

---

## Por que twitterapi.io

| | Twitter/X Oficial (Basic) | twitterapi.io |
|---|---|---|
| Custo | $100/mês | ~$2/mês (pay-as-you-go) |
| Aprovação | Necessária | Não necessária |
| Autenticação | OAuth 2.0 complexo | API Key simples |
| Busca de tweets | Incluída | Incluída |
| Latência | <500ms | <500ms |

---

## Configuração

### 1. Criar conta

1. Acesse [twitterapi.io](https://twitterapi.io)
2. Clique em **Get Started** → crie a conta
3. Você recebe **$0.10 de crédito gratuito** sem cartão de crédito

### 2. Copiar a API Key

1. No painel, acesse **API Keys**
2. Copie a chave gerada

### 3. Adicionar variável no n8n

| Variável | Valor |
|---|---|
| `TWITTERAPI_IO_KEY` | sua API key |

---

## Endpoint utilizado

```
GET https://api.twitterapi.io/twitter/tweet/advanced_search
```

**Header de autenticação:**
```
X-API-Key: <TWITTERAPI_IO_KEY>
```

**Parâmetros enviados:**

| Parâmetro | Valor |
|---|---|
| `query` | query de busca (ver abaixo) |
| `queryType` | `Latest` — tweets mais recentes primeiro |

---

## Query de busca padrão

```
(breaking OR "just in") (election OR rates OR fed OR crypto OR bitcoin OR trump OR tariff OR ceasefire OR war) -is:retweet lang:en
```

### Customizar por tema

**Política EUA:**
```
(breaking OR "just in") (trump OR congress OR senate OR "supreme court" OR tariff) -is:retweet lang:en
```

**Cripto:**
```
(breaking OR "just in") (bitcoin OR ethereum OR crypto OR sec OR etf OR halving) -is:retweet lang:en
```

**Geopolítica:**
```
(breaking OR "just in") (war OR ceasefire OR sanctions OR nato OR ukraine OR israel) -is:retweet lang:en
```

---

## Estrutura da resposta

```json
{
  "tweets": [
    {
      "id": "1234567890",
      "text": "BREAKING: Fed signals rate cut in September...",
      "url": "https://x.com/user/status/1234567890",
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
  "next_cursor": "cursor_token"
}
```

**Campos utilizados no workflow:**

| Campo | Uso |
|---|---|
| `text` | Fonte da notícia |
| `id` | Identificação do tweet |
| `url` | Link para o tweet original |
| `createdAt` | Filtro de recência (> 35 min → descartado) |
| `author.userName` | Exibido no alerta Telegram (`@Reuters`) |
| `author.followers` | Bônus de confiança (+5% se > 100k seguidores) |

---

## Custo estimado

| Cenário | Cálculo | Custo/mês |
|---|---|---|
| Schedule 30 min / 10 tweets | 48 × 10 × 30 = 14.400 tweets | **~$2,16** |
| Schedule 15 min / 10 tweets | 96 × 10 × 30 = 28.800 tweets | **~$4,32** |

Preço: $0,15 por 1.000 tweets.

---

## Troubleshooting

| Erro | Causa | Solução |
|---|---|---|
| `401 Unauthorized` | API Key inválida | Verifique a chave no painel twitterapi.io |
| Array `tweets: []` vazio | Query sem resultados recentes | Amplie os termos da query |
| Tweets irrelevantes | Query ampla demais | Adicione termos mais específicos |
| `429 Too Many Requests` | Rate limit atingido | Aumente o intervalo do schedule |
