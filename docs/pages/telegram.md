# Integração com Telegram

---

## Configuração do Bot

### 1. Criar o bot

No Telegram, inicie conversa com [@BotFather](https://t.me/BotFather):

```
/newbot
→ Nome do bot: Polymarket Alert Bot
→ Username: polymarket_alert_bot (deve terminar em "bot")
→ Token gerado: 123456789:AABBccDDeeFFggHH...
```

### 2. Obter o Chat ID

**Opção A — Chat privado:**
1. Inicie conversa com o bot
2. Envie qualquer mensagem
3. Acesse: `https://api.telegram.org/bot<TOKEN>/getUpdates`
4. Copie o valor de `message.chat.id`

**Opção B — Grupo:**
1. Adicione o bot ao grupo
2. Envie uma mensagem no grupo
3. Acesse a URL acima
4. O Chat ID de grupos começa com `-100`

---

## Formato da Mensagem

O node `Gerar Sinal` constrói a mensagem com **HTML** do Telegram:

```
🚨 ALERTA DE TRADE - POLYMARKET

🐦 Tweet de @Reuters:
Federal Reserve mantém taxa de juros e sinaliza...

📊 Mercado:
Will the Fed cut interest rates in September 2026?

🎯 Direcao: BUY_YES
💪 Confianca: 78%
📈 Movimento: 22% → 42%
💰 Volume: $125,000
🤖 Query IA: fed rate cut

🔗 https://polymarket.com/event/fed-cut-rates-september-2026
```

---

## Elementos de Formatação

| Elemento | HTML usado | Renderiza como |
|---|---|---|
| Título | `<b>texto</b>` | **negrito** |
| Direção / Query IA | `<code>BUY_YES</code>` | `código monoespaçado` |
| Labels | `<b>Label:</b> valor` | **Label:** valor |

> O parse mode é `HTML`. Todo conteúdo dinâmico (tweet, título do mercado, autor, query) é escapado via função `esc()` que substitui `&`, `<`, `>` pelos respectivos HTML entities, evitando erros de parsing.

---

## Node Telegram Alert

| Parâmetro | Valor |
|---|---|
| Resource | `message` |
| Operation | `sendMessage` |
| Chat ID | hardcoded no JSON |
| Text | `={{ $json.telegram_message }}` |
| Parse Mode | `HTML` |

**Credencial:** `Telegram Bot API` — configurada em **n8n → Settings → Credentials**.

---

## Exemplo de Resposta da API Telegram

Quando o envio é bem-sucedido, o node retorna:

```json
{
  "ok": true,
  "result": {
    "message_id": 42,
    "chat": {
      "id": 2074413015,
      "type": "private"
    },
    "date": 1743764400,
    "text": "🚨 ALERTA DE TRADE - POLYMARKET\n\n..."
  }
}
```

---

## Casos de Erro Comuns

| Erro Telegram | Causa | Solução |
|---|---|---|
| `401 Unauthorized` | Token inválido | Verifique o token gerado pelo BotFather |
| `400 Bad Request: chat not found` | Chat ID inválido | Confirme o ID via `getUpdates` |
| `403 Forbidden` | Bot removido do grupo | Re-adicione o bot ao grupo |
| `400 can't parse entities` | HTML inválido no conteúdo do tweet | Verificar se `esc()` está sendo aplicado em todos os campos dinâmicos |

---

## Personalizar a Mensagem

Para alterar o texto do alerta, edite a variável `msg` no node `Gerar Sinal`:

```javascript
const msg = "🚨 <b>ALERTA DE TRADE - POLYMARKET</b>\n\n"
  + "🐦 <b>Tweet de " + authorInfo + ":</b>\n" + newsSummary + "\n\n"
  + "📊 <b>Mercado:</b>\n" + question + "\n\n"
  + "🎯 <b>Direcao:</b> <code>" + direction + "</code>\n"
  + "💪 <b>Confianca:</b> " + confidence + "%\n"
  + "📈 <b>Movimento:</b> " + yesPct + "% → " + expectedTo + "%\n"
  + "💰 <b>Volume:</b> $" + Math.round(volume).toLocaleString("en-US") + aiQueryInfo + "\n\n"
  + "🔗 " + marketUrl;
```

> Sempre aplique `esc()` em qualquer novo campo dinâmico adicionado à mensagem.
