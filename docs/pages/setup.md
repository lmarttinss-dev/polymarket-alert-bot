# Configuração e Instalação

---

## Pré-requisitos

- n8n instalado (self-hosted ou cloud)
- Conta no [twitterapi.io](https://twitterapi.io)
- Conta no [Google AI Studio](https://aistudio.google.com) com créditos
- Bot do Telegram criado via [@BotFather](https://t.me/BotFather)

---

## 1. Configurar twitterapi.io

1. Acesse [twitterapi.io](https://twitterapi.io) → **Get Started**
2. Crie a conta (recebe $0.10 de crédito grátis)
3. No painel, acesse **API Keys** e copie a chave

> A chave está hardcoded no JSON do workflow. Atualize o campo `X-API-Key` no node **Buscar Tweets** se precisar trocar a chave.

---

## 2. Configurar Google Gemini

1. Acesse [aistudio.google.com](https://aistudio.google.com)
2. Clique em **Get API Key** → **Create API Key**
3. Copie a chave gerada

### Credencial no n8n

1. **Settings → Credentials → Add Credential → Google Gemini (PaLm) API**
2. Cole a **API Key**
3. Salve como `Google Gemini API`

---

## 3. Configurar Telegram

### 3.1 Criar o bot

No Telegram, converse com [@BotFather](https://t.me/BotFather):

```
/newbot
→ Nome: Polymarket Alert Bot
→ Username: qualquer_nome_bot
→ Token: 123456789:AABBccDDeeFF...
```

### 3.2 Obter o Chat ID

1. Inicie conversa com o bot → clique **Start** → envie qualquer mensagem
2. Acesse no navegador:
```
https://api.telegram.org/bot<TOKEN>/getUpdates
```
3. Copie o valor de `message.chat.id`

> O Chat ID está hardcoded no JSON do workflow. Atualize o campo `chatId` no node **Telegram Alert** se precisar trocar.

### 3.3 Credencial no n8n

1. **Settings → Credentials → Add Credential → Telegram**
2. Cole o **Access Token** do BotFather
3. Salve como `Telegram Bot API`

---

## 4. Importar o Workflow

1. n8n → **Workflows → Add Workflow → Import from File**
2. Selecione `polymarket_trade_alert.json`
3. O workflow importa com todos os nodes conectados

---

## 5. Vincular Credenciais

Após importar, vincule as credenciais nos nodes:

| Node | Credencial |
|---|---|
| **Agente de IA** → sub-nó **Google Gemini Chat Model** | `Google Gemini API` |
| **Telegram Alert** | `Telegram Bot API` |

---

## 6. Ativar

Toggle **Active** no canto superior direito → verde.

O workflow passa a executar automaticamente a cada 30 minutos.

---

## 7. Testar Manualmente

1. Abra o workflow no editor
2. Clique em **Execute Workflow**
3. Acompanhe node por node
4. Verifique o Telegram se houver sinal `TRADE`

---

## Troubleshooting

| Problema | Causa | Solução |
|---|---|---|
| `401` no Buscar Tweets | API Key do twitterapi.io inválida | Atualize o campo `X-API-Key` no node |
| `Bad request` no Gemini | API Key inválida ou sem créditos | Verifique em aistudio.google.com → Billing |
| `NO_TRADE` em todos os tweets | LLM considera irrelevantes ou sem mercados na Polymarket | Normal para tweets sem relação com mercados de predição |
| Telegram silencioso | Chat ID ou credencial incorreta | Valide via `getUpdates` |
| Loop não termina | Conexão de retorno faltando | Confirme que `Telegram Alert` e `Sem Oportunidade` conectam ao `Processar um por um` |
| Mesmo link sempre | Mercados genéricos passando no filtro | Verificar se a `ai_query` é específica o suficiente |
