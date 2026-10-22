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

### Etapa 4 — Gráficos (`04_graficos.ipynb`)
- Quatro gráficos em matplotlib, salvos em `figuras/`: população mundial, os 5 países mais populosos, crescimento por continente e as maiores variações até 2030.
- Paleta validada para daltonismo, números no padrão brasileiro e textos em cinza (a cor fica só nas linhas e barras).

### Etapa 5 — Crescimento e idosos (`05_idosos.ipynb`)
- **Pergunta:** quais países mais cresceram e mais encolheram entre 2000 e 2022, e se a população idosa cresceu.
- **Por que uma fonte externa:** a base original só tem a população total, sem divisão por idade. Os dados de 65+ vêm do Banco Mundial (`SP.POP.65UP.TO` e `SP.POP.65UP.TO.ZS`), ligados à base pelo código ISO3 (`cca3`). 214 dos 234 países e territórios têm dados.
- **Ranking (países com 1 mi+):** os que mais cresceram foram o Catar (+317%) e os Emirados (+188%), por imigração de trabalhadores, e Níger, Angola e Chade, por natalidade alta. Os que mais encolheram foram Lituânia, Moldávia e Letônia (cerca de −23%), por emigração de jovens somada à baixa natalidade.
- **Resposta: sim, a população idosa cresceu.** No mundo, passou de 421 para 778 milhões de pessoas (de 6,8% para 9,7% da população). O número de idosos aumentou em 213 de 214 países (a exceção é o Lesoto, afetado pelo HIV/aids), e a fatia de idosos aumentou em 191 (89%).
- Quem mais encolhe é quem mais envelhece: a mediana de idosos é de 20,4% nos que mais encolheram, contra 2,7% nos que mais cresceram.

### Merge na `main`
- A branch `projeto-forecast` foi juntada na `main` por fast-forward (commit `5e35590`), sem conflitos: a `main` não tinha mudado desde a criação da branch. Antes da troca, a prévia do resultado foi revisada.
- Depois do merge, as branches `projeto-forecast` (no GitHub e local) e `falha-forecast-2026-10-22` foram apagadas.

### Etapa 6 — Automação (agente agendado do Claude)
- **Ferramenta escolhida:** um agente agendado do Claude (routine na nuvem), e não GitHub Actions. Além de executar, o agente confere os resultados e explica o que mudou.
- **Frequência:** quinzenal, nos **dias 1 e 15 de cada mês às 8h** (horário de Brasília; cron `0 11 1,15 * *` em UTC). O CSV principal é fixo e os dados do Banco Mundial mudam pouco, então rodar todo dia geraria quase sempre o mesmo resultado.
- **Modelo:** `claude-sonnet-5-5`, no ambiente `Default`.
- **Painel:** https://claude.ai/code/routines/trig_01CSxbMeYWTt3bn1ntT8wspv
- **O que o agente faz:**
  1. Clona a `main` e instala as bibliotecas.
  2. Apaga o CSV de idosos para forçar o download atualizado do Banco Mundial.
  3. Roda os notebooks 01 → 05 em ordem.
  4. Confere os resultados: verificações dos notebooks, população mundial de 2030 entre 8,3 e 8,9 bi, anos novos ou revisões no Banco Mundial, e mudanças nos rankings.
- **Regras de saída:**
  - **Sem mudança nos dados:** não faz commit nem PR. Diferenças só de horário de execução ou de renderização dos PNGs não contam como mudança.
  - **Dados novos:** cria a branch `atualizacao-forecast-AAAA-MM-DD`, registra aqui no histórico o "antes → depois" e abre um **PR para revisão**.
  - **Falha:** cria a branch `falha-forecast-AAAA-MM-DD`, que só descreve o erro no histórico, e abre um PR, sem corrigir dados nem alterar notebooks.
  - O agente nunca envia nada direto para a `main`.
- **Configuração necessária:**
  - conexão da conta do GitHub com o Claude (app do Claude instalado só no repositório `cienciadedados`);
  - liberação de `api.worldbank.org` na rede do ambiente `Default`.
- **Testes em 22/10/2026:**

  | Teste | Resultado | Causa | Correção |
  |---|---|---|---|
  | 1º | ❌ O 05 falhou e o PR não foi aberto | Rede bloqueando o Banco Mundial; app sem permissão de escrita | Liberar o domínio; selecionar o repositório no app |
  | 2º | ❌ O 05 falhou, mas o PR de falha #4 foi aberto | GitHub resolvido, rede ainda bloqueada | Liberar `api.worldbank.org` |
  | 3º | ✅ "Sem mudanças nos dados" | Os 5 notebooks rodaram, 2030 = 8,600 bi e os dados do Banco Mundial estavam idênticos | — |

  O PR #4 foi fechado sem merge, por ser apenas o registro de um teste.
- **Primeira execução oficial:** 1º/11/2026 às 8h.

### Pendências
- **Descrição do repositório no GitHub:** atualizar manualmente, pela engrenagem do "About", porque ela ainda fala só de Data Warehouse.

## Próximos passos
1. Acompanhar a primeira execução automática (1º/11) e revisar os PRs que o agente abrir.
2. Gráficos do envelhecimento (opcional).
