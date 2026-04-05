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

```
yesPrice = outcomePrices[0]   // preço atual do YES (0.0 a 1.0)
yesPct   = round(yesPrice * 100)
```

| Condição | Direção | Lógica |
|---|---|---|
| `yesPrice <= 0.30` | `BUY_YES` | Mercado subprecificado — notícia aumenta chance de YES |
| `yesPrice >= 0.70` | `BUY_NO` | Mercado superprecificado — notícia diminui chance de YES |
| `0.31 <= yesPrice <= 0.69` | `BUY_YES` | Zona neutra — assume viés de alta com menor confiança |

---

## Cálculo de Confiança

### Base

| Condição de entrada | Confiança base |
|---|---|
| `yesPrice <= 0.30` | 70% |
| `yesPrice >= 0.70` | 68% |
| `0.31–0.69` (zona neutra) | 60% |

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

O movimento esperado é o range de preço projetado pelo sistema, dado em pontos percentuais de probabilidade.

| Direção | Fórmula | Cap |
|---|---|---|
| `BUY_YES` (yesPrice ≤ 0.30) | `expectedTo = yesPct + 20` | máx 95% |
| `BUY_YES` (zona neutra) | `expectedTo = yesPct + 12` | máx 95% |
| `BUY_NO` | `expectedTo = yesPct - 20` | mín 5% |

**Exemplo na mensagem Telegram:**
```
Movimento: 22% → 42%
```
Significa: o YES está em 22% e o sistema projeta que chegue a 42%.

---

## Fluxo de Decisão Completo

```
                    ┌──────────────────┐
                    │  yesPrice atual  │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
         ≤ 30%          31%–69%          ≥ 70%
              │              │              │
          BUY_YES        BUY_YES        BUY_NO
          base=70%       base=60%       base=68%
          +20pts         +12pts         -20pts
              │              │              │
              └──────────────┼──────────────┘
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

## Limitações do MVP

A lógica de direção é determinística e baseada apenas no preço atual do YES.

**Melhorias recomendadas para produção:**

| Melhoria | Descrição |
|---|---|
| Validação LLM do mercado | Perguntar ao LLM se o mercado encontrado realmente corresponde à notícia antes de gerar sinal |
| Análise de sentimento | Usar o LLM para detectar se a notícia é positiva ou negativa para o YES do mercado |
| Dados históricos de preço | Comparar com volatilidade passada do mercado |
| Spread bid/ask | Incorporar o spread real para calcular o ponto de breakeven |
| Stop loss automático | Definir nível de saída com base no risco |
| Multi-mercado | Analisar os top 5 mercados retornados, não apenas o primeiro |
