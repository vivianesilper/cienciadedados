# Projeto Forecast

Previsão populacional por país a partir da base *World Population* (populações de 1970 a 2022 para 234 países e territórios).

## Etapas

| Notebook | Descrição |
|---|---|
| [`01_ingestao.ipynb`](01_ingestao.ipynb) | Carrega o CSV, inspeciona estrutura, nulos e duplicados |
| [`02_limpeza.ipynb`](02_limpeza.ipynb) | Corrige `Density` e `Growth Rate`, padroniza colunas e exporta a base limpa |
| [`03_forecast.ipynb`](03_forecast.ipynb) | Backtest de métodos (treino até 2015, teste em 2020/2022) e previsão por país para 2025 e 2030 |

## Resultados

- Método escolhido: tendência linear entre os dois últimos pontos observados (erro mediano de 1,75% no teste de 2022, contra 7,96% da previsão ingênua).
- População mundial prevista: **8,21 bi em 2025** e **8,60 bi em 2030**.

## Dados

- `data/world_population.csv`: base original (separador `;`)
- `data/world_population_limpo.csv`: base tratada, gerada pelo `02_limpeza.ipynb`
- `data/forecast_populacao.csv`: previsões para 2025 e 2030, geradas pelo `03_forecast.ipynb`

## Como executar

```bash
pip install pandas jupyter
jupyter notebook
```

Execute os notebooks em ordem, a partir da pasta `Projeto_Forecast`.
