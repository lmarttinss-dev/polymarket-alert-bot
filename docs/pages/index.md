# Polymarket Trade Alert Bot

Automação autônoma de detecção de oportunidades de trade em mercados de previsão. Monitora breaking news no Twitter/X via twitterapi.io a cada 30 minutos, usa LLM (Google Gemini) para avaliar relevância e gerar queries de busca, consulta mercados ativos na Polymarket e envia alertas via Telegram quando uma oportunidade é identificada.

---

## Visão Geral

```
[Schedule: a cada 30 min]
        │
        ▼
[twitterapi.io — Buscar breaking news]
        │
        ▼
[Extrair Tweets — normalizar array]
        │
        ▼
[Split In Batches — 1 tweet por vez] ◄──────────────────────┐
        │                                                    │
        ▼                                                    │
[Analisar Noticia — filtrar por recência]                    │
        │                                                    │
        ▼                                                    │
[É Recente? — IF: sem NO_TRADE]                              │
   ┌────┴────┐                                               │
  SIM       NÃO → [Sem Oportunidade] ──────────────────────►│
   │                                                         │
   ▼                                                         │
[Preparar Prompt IA — montar prompt]                         │
        │                                                    │
        ▼                                                    │
[Agente de IA — Gemini 2.5 Flash]                            │
        │                                                    │
        ▼                                                    │
[Extrair Query IA — parsear resposta LLM]                    │
        │                                                    │
        ▼                                                    │
[É Relevante? — IF: sem NO_TRADE]                            │
   ┌────┴────┐                                               │
  SIM       NÃO → [Sem Oportunidade] ──────────────────────►│
   │                                                         │
   ▼                                                         │
[Polymarket API — buscar mercados]                           │
        │                                                    │
        ▼                                                    │
[Filtrar — volume > $500, match ≥ 2 palavras, top 5]        │
        │                                                    │
        ▼                                                    │
[Gerar Sinal — BUY_YES / BUY_NO]                             │
        │                                                    │
        ▼                                                    │
[Verificar Oportunidade]                                     │
   ┌────┴────┐                                               │
  SIM       NÃO                                              │
   │         │                                               │
[Telegram] [Sem Oportunidade] ──────────────────────────────┘
```

---

## Stack

| Componente | Tecnologia |
|---|---|
| Orquestração | n8n |
| Fonte de notícias | twitterapi.io (proxy Twitter/X) |
| LLM | Google Gemini 2.5 Flash |
| Mercados de previsão | Polymarket Gamma API |
| Notificações | Telegram Bot API |
| Linguagem dos nodes | JavaScript (ES2020) |

---

## Páginas desta documentação

| Página | Descrição |
|---|---|
| [Arquitetura](./architecture.md) | Diagrama completo do fluxo e loop de batches |
| [Nodes](./nodes.md) | Referência técnica dos nodes |
| [Twitter/X](./twitter.md) | twitterapi.io — configuração, query, custos |
| [Configuração](./setup.md) | Passo a passo para importar e ativar |
| [Lógica de Sinal](./signal-logic.md) | Como BUY_YES/BUY_NO e confiança são calculados |
| [API Reference](./api-reference.md) | Contratos das APIs externas consumidas |
| [Telegram](./telegram.md) | Formato da mensagem e configuração do bot |

---

## Credenciais necessárias no n8n

| Credencial | Tipo | Onde obter |
|---|---|---|
| Telegram Bot Token | Telegram (n8n built-in) | @BotFather no Telegram |
| Google Gemini API Key | Google Gemini (PaLm) API | aistudio.google.com |
