# Histórico do Projeto Forecast

Registro das decisões tomadas ao longo do projeto: o que foi feito, por quê e o que ficou para depois.

## 22/10/2026 — Início do projeto

### Organização
- **Base escolhida:** `world_population.csv` (234 países e territórios, populações de 1970 a 2022).
- **Repositório:** o projeto foi primeiro salvo no repositório `.jupyter`, mas foi movido para o `cienciadedados`. O motivo: `.jupyter` é o nome da pasta de configuração do Jupyter e não comunica o conteúdo, enquanto `cienciadedados` funciona como portfólio. O `.jupyter` ficou como estava.
- **Branch:** o trabalho é feito na branch `projeto-forecast` e entra na `main` por pull request.
- **Pasta:** o projeto fica em `Projeto_Forecast/`, separado do conteúdo de Data Warehouse que já existia no repositório. Pasta e branch são coisas independentes: a pasta não afeta pull requests.
- **Pastas locais:** os commits são feitos em `C:\Users\Viviane Silper\cienciadedados\Projeto_Forecast`, e uma cópia idêntica é mantida em `C:\Users\Viviane Silper\.jupyter\Projeto_Forecast`.

### Etapa 1 — Ingestão (`01_ingestao.ipynb`)
- O CSV usa `;` como separador.
- Não há valores nulos nem duplicados.
- **Problema encontrado:** duas colunas perderam a vírgula decimal na exportação do arquivo:
  - `Density (per km²)`: `630.587` em vez de `63,06`, e `2.224.774` em vez de `222,48`. Por isso a coluna é lida como texto.
  - `Growth Rate`: `10.257` em vez de `1.0257`.
  - `World Population Percentage` está correta (a soma dá 99,93%).

### Etapa 2 — Limpeza (`02_limpeza.ipynb`)
- **Densidade:** recalculada como `pop_2022 ÷ área`. Não dá para recuperar o valor a partir do texto corrompido com segurança, mas o recálculo confere com os poucos valores que vieram corretos.
- **Growth Rate:** os valores ≥ 10 foram divididos por 10. A separação é clara: não existe nenhum valor entre 1,1 e 9. Depois da correção, todos ficam entre 0,91 e 1,07.
- **Colunas:** os nomes foram padronizados em `snake_case` (`pop_2022`, `densidade_km2`, `taxa_crescimento`...).
- **Saída:** `data/world_population_limpo.csv`.

### Etapa 3 — Forecast (`03_forecast.ipynb`)
- **Backtest:** treino com os dados até 2015 e teste nos valores reais de 2020 e 2022.

  | Método | Erro mediano 2022 | p90 2022 |
  |---|---|---|
  | **Linear, últimos 2 pontos** (escolhido) | **1,75%** | 7,16% |
  | Exponencial, últimos 2 pontos | 1,87% | 7,88% |
  | Linear, últimos 3 pontos | 2,91% | 8,31% |
  | Ingênuo (população constante) | 7,96% | 16,82% |

- **Por que 2 pontos:** usar mais pontos mistura as tendências das décadas de 1990 e 2000 com a dinâmica recente, e o erro aumenta.
- **Maiores erros:** Líbano, Síria e países do Golfo, que passaram por guerras e mudanças bruscas de migração. Nenhum modelo baseado só no histórico consegue captar isso.
- **Previsão final:** reta entre **2015 e 2022**. O intervalo de 7 anos é parecido com o do backtest (5 anos), e evita depender só de 2020→2022, período afetado pela pandemia.
- **Ano a ano:** de 2022 (real) a 2027, com o crescimento % de cada ano.

  | | 2022 | 2023 | 2024 | 2025 | 2026 | 2027 | 2030 |
  |---|---|---|---|---|---|---|---|
  | Mundo (bi) | 7,97 | 8,05 | 8,13 | 8,21 | 8,29 | 8,37 | 8,60 |

- **Conferência:** a previsão de 2023 bate com a `taxa_crescimento` da própria base (diferença mediana de 0,17% por país). O total de 2030 está próximo da projeção da ONU (~8,5 bi).
- **Saídas:** `data/forecast_populacao.csv` (2025 e 2030) e `data/forecast_anual_2022_2027.csv`.

### Decisões adiadas
- **Automação (pipeline diária):** fica para o final do projeto, quando soubermos exatamente o que rodar. Opções levantadas: GitHub Actions + Papermill (recomendada), agente agendado do Claude, Prefect ou Airflow, Kaggle Notebooks e o Agendador de Tarefas do Windows. Observação: com o CSV fixo, rodar todo dia gera sempre o mesmo resultado. A automação só faz sentido com uma fonte de dados que se atualiza ou como teste de que os notebooks continuam funcionando.
- **Descrição do repositório no GitHub:** precisa ser atualizada manualmente, pela engrenagem do "About".

## Próximos passos
1. Gráficos: série histórica com a previsão dos maiores países e o crescimento por continente.
2. Pull request da branch `projeto-forecast` para a `main`.
3. Automação da pipeline.
