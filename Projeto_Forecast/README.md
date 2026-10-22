# Projeto Forecast

Previsão populacional por país a partir da base *World Population* (populações de 1970 a 2022 para 234 países e territórios).

## Etapas

| Notebook | Descrição |
|---|---|
| [`01_ingestao.ipynb`](01_ingestao.ipynb) | Carrega o CSV, inspeciona estrutura, nulos e duplicados |
| [`02_limpeza.ipynb`](02_limpeza.ipynb) | Corrige `Density` e `Growth Rate`, padroniza colunas e exporta a base limpa |

## Dados

- `data/world_population.csv`: base original (separador `;`)
- `data/world_population_limpo.csv`: base tratada, gerada pelo `02_limpeza.ipynb`

## Como executar

```bash
pip install pandas jupyter
jupyter notebook
```

Execute os notebooks em ordem, a partir da pasta `Projeto_Forecast`.
