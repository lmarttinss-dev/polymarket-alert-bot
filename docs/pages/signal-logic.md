# Lógica de Sinal de Trade

Descrição completa de como o sistema decide a direção, confiança e movimento esperado para cada oportunidade detectada.

---

## Conceitos do Polymarket

No Polymarket, cada mercado tem uma pergunta binária (ex: "O Fed vai cortar juros em setembro?") com dois resultados possíveis: **YES** e **NO**.

O preço de cada resultado representa a probabilidade implícita do mercado:
- Preço YES = 0.25 → mercado acredita que há 25% de chance de SIM
- Preço YES = 0.80 → mercado acredita que há 80% de chance de SIM

Preço YES + Preço NO ≈ 1.00 (desconsiderando spread).

---

## Seleção do Mercado

O LLM (Gemini 2.5 Flash) gera uma query de busca específica para a notícia. O node `Filtrar Mercados` retorna os mercados que:
1. Têm `volume > $500`
2. Contêm **pelo menos 2 palavras** da query no título ou descrição
3. Estão ordenados por volume decrescente

O node `Gerar Sinal` usa **sempre o primeiro** da lista (maior volume), que representa o mercado mais líquido e relevante.

---

## Regras de Direção

A direção é determinada pela **análise de sentimento do LLM** (n22–n25): o Gemini avalia se a notícia é positiva ou negativa para o resultado YES do mercado.

```
yesPrice = outcomePrices[0]   // preço atual do YES (0.0 a 1.0)
yesPct   = round(yesPrice * 100)
sentiment = "POSITIVE" | "NEGATIVE"   // saída do LLM (n25)
```

| Sentimento LLM | Direção | Lógica |
|---|---|---|
| `POSITIVE` | `BUY_YES` | A notícia aumenta a probabilidade de YES ser correto |
| `NEGATIVE` | `BUY_NO` | A notícia diminui a probabilidade de YES ser correto |

> O sentimento é avaliado especificamente para a pergunta do mercado encontrado, não de forma genérica. O LLM recebe o tweet e a pergunta do mercado e decide: "Esta notícia torna mais ou menos provável que o YES desta pergunta se concretize?"

---

## Cálculo de Confiança

### Base

A confiança base ainda usa o preço atual do YES para calibrar a magnitude — preços extremos indicam maior convicção do mercado, validando o sinal.

| Condição de entrada | Confiança base |
|---|---|
| Sentimento `POSITIVE` + `yesPrice <= 0.30` | 70% |
| Sentimento `NEGATIVE` + `yesPrice >= 0.70` | 68% |
| Sentimento qualquer, preço na zona neutra | 60% |

### Bônus por Volume

Volume alto indica mercado líquido com preço mais eficiente, o que valida o sinal.

| Volume do mercado | Bônus |
|---|---|
| > $100.000 | +8% |
| > $500.000 | +5% adicional |
| Máximo absoluto | 88% |

### Bônus por Seguidores do Autor

| Seguidores | Bônus |
|---|---|
| > 100.000 | +5% |

**Exemplo:**
```
yesPrice = 0.22  →  base = 70%
volume   = $250.000  →  +8%
author_followers = 24.000.000  →  +5%
total    = 83%  (cap: 88%)
```

---

## Cálculo do Movimento Esperado

O movimento esperado é o range de preço projetado pelo sistema, dado em pontos percentuais de probabilidade. A magnitude continua calibrada pelo preço atual, independente do sentimento.

| Direção | Condição de preço | Fórmula | Cap |
|---|---|---|---|
| `BUY_YES` | `yesPrice <= 0.30` | `expectedTo = yesPct + 20` | máx 95% |
| `BUY_YES` | zona neutra | `expectedTo = yesPct + 12` | máx 95% |
| `BUY_NO` | `yesPrice >= 0.70` | `expectedTo = yesPct - 20` | mín 5% |
| `BUY_NO` | zona neutra | `expectedTo = yesPct - 12` | mín 5% |

**Exemplo na mensagem Telegram:**
```
Movimento: 22% → 42%
```
Significa: o YES está em 22% e o sistema projeta que chegue a 42%.

---

## Fluxo de Decisão Completo

```
                    ┌──────────────────┐
                    │ Sentimento LLM   │  n25 — POSITIVE ou NEGATIVE
                    └────────┬─────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
          POSITIVE                      NEGATIVE
              │                             │
          BUY_YES                       BUY_NO
              │                             │
     ┌────────┴────────┐         ┌──────────┴──────────┐
  ≤30% → +20pts      else        ≥70% → -20pts       else
  base=70%         +12pts        base=68%           -12pts
                   base=60%                         base=60%
              │                             │
              └──────────────┬──────────────┘
                             │
                    ┌────────┴────────┐
                    │  Bônus Volume   │
                    │  >100k: +8%     │
                    │  >500k: +5%     │
                    │  máx: 88%       │
                    └────────┬────────┘
                             │
                    ┌────────┴────────┐
                    │ Bônus Seguidores│
                    │  >100k: +5%     │
                    └────────┬────────┘
                             │
                    ┌────────┴────────┐
                    │  Sinal Final    │
                    │  direction      │
                    │  confidence     │
                    │  expected_from  │
                    │  expected_to    │
                    └─────────────────┘
```

---

## Melhorias Futuras

**Implementadas:**
- Validação LLM do mercado (n18–n21) — confirma relevância antes de gerar sinal
- Análise de sentimento LLM (n22–n25) — detecta direção POSITIVE/NEGATIVE automaticamente

**Recomendadas para produção:**

| Melhoria | Descrição |
|---|---|
| Dados históricos de preço | Comparar com volatilidade passada do mercado |
| Spread bid/ask | Incorporar o spread real para calcular o ponto de breakeven |
| Stop loss automático | Definir nível de saída com base no risco |
| Multi-mercado | Analisar os top 5 mercados retornados, não apenas o primeiro |
