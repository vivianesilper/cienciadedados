# Dashboard no Power BI

Passo a passo para montar o dashboard do Projeto Forecast a partir dos arquivos Parquet gerados pelo [`06_powerbi.ipynb`](06_powerbi.ipynb) em `data/powerbi/`.

## 1. Tabelas

| Tabela | Grão | Colunas principais |
|---|---|---|
| `dim_pais` | país | `cca3` (chave), `pais`, `capital`, `continente`, `area_km2`, `densidade_km2`, `rank_pop_2022`, `pct_pop_mundial_2022` |
| `dim_ano` | ano (1970–2030) | `ano` (chave), `decada`, `periodo` (Histórico/Previsão) |
| `fato_populacao` | país × ano | `cca3`, `ano`, `tipo` (Histórico/Previsão), `populacao` |
| `fato_idosos` | país × ano (1970–2024) | `cca3`, `ano`, `idosos_65mais`, `pct_idosos_65mais`, `populacao_bm` |
| `fato_crescimento_idosos` | país | `cca3`, `pop_2000`, `pop_2022`, `idosos_2000`, `idosos_2022`, `pct_idosos_2000`, `pct_idosos_2022`, `cresc_pop_pct`, `cresc_idosos_pct`, `var_pct_idosos_pp` |

Anos históricos de `fato_populacao`: 1970, 1980, 1990, 2000, 2010, 2015, 2020 e 2022. Anos previstos: 2023 a 2027 e 2030.

## 2. Importação

1. **Página Inicial → Transformar dados → Gerenciar parâmetros → Novo parâmetro**:
   - Nome: `PastaDados`
   - Tipo: Texto
   - Valor: o caminho da pasta, terminado em `\`. Exemplo: `C:\Users\Viviane Silper\cienciadedados\Projeto_Forecast\data\powerbi\`
2. Para cada uma das 5 tabelas, siga **Nova fonte → Consulta nula → Editor avançado** e cole o código abaixo, trocando o nome do arquivo:

   ```m
   let
       Fonte = Parquet.Document(File.Contents(PastaDados & "dim_pais.parquet"))
   in
       Fonte
   ```

3. Renomeie cada consulta com o nome do arquivo (`dim_pais`, `dim_ano`, …) e clique em **Fechar e aplicar**.

Com o parâmetro, quando os notebooks forem rodados de novo (por exemplo, depois de um PR do agente quinzenal), basta clicar em **Atualizar**. Se a pasta mudar de lugar, altere só o valor do parâmetro.

## 3. Modelo (relacionamentos)

Na exibição **Modelo**, crie os relacionamentos abaixo, todos **muitos para um (*:1)** e com **filtro em direção única**:

| De (muitos) | Para (um) |
|---|---|
| `fato_populacao[cca3]` | `dim_pais[cca3]` |
| `fato_idosos[cca3]` | `dim_pais[cca3]` |
| `fato_crescimento_idosos[cca3]` | `dim_pais[cca3]` |
| `fato_populacao[ano]` | `dim_ano[ano]` |
| `fato_idosos[ano]` | `dim_ano[ano]` |

Ajustes recomendados:

- Oculte as colunas `cca3` e `ano` das tabelas fato (botão direito → *Ocultar na exibição de relatório*). Assim, os filtros e eixos usam sempre as dimensões, e um mesmo filtro vale para todas as fatos.
- Em `dim_pais[pais]`, mude a **Categoria de dados** para *País/Região*, para o mapa reconhecer os nomes.
- Em `dim_ano[ano]`, desative o resumo (*Não resumir*).
- Crie uma tabela vazia `_Medidas` (Inserir dados → OK) para guardar as medidas abaixo.

## 4. Medidas DAX

```dax
População = SUM ( fato_populacao[populacao] )

População Histórica =
CALCULATE ( [População], KEEPFILTERS ( fato_populacao[tipo] = "Histórico" ) )

-- inclui 2022 para a linha da previsão sair do último ponto real, sem buraco no gráfico
População Prevista =
CALCULATE (
    [População],
    KEEPFILTERS ( fato_populacao[tipo] = "Previsão" || fato_populacao[ano] = 2022 )
)

População 2022 = CALCULATE ( [População], dim_ano[ano] = 2022 )
População 2030 = CALCULATE ( [População], dim_ano[ano] = 2030 )
Variação 2022–2030 % = DIVIDE ( [População 2030] - [População 2022], [População 2022] )

-- os anos históricos têm intervalos (1970, 1980, ...); o "ano anterior" é o último ano com dado
População Ano Anterior =
VAR AnoAtual = MAX ( dim_ano[ano] )
VAR AnoAnterior =
    CALCULATE ( MAX ( fato_populacao[ano] ), ALL ( dim_ano ), fato_populacao[ano] < AnoAtual )
RETURN
    CALCULATE ( [População], ALL ( dim_ano ), dim_ano[ano] = AnoAnterior )

Crescimento vs Ano Anterior % = DIVIDE ( [População] - [População Ano Anterior], [População Ano Anterior] )

Idosos 65+ = SUM ( fato_idosos[idosos_65mais] )

-- ponderado: soma dos idosos ÷ soma da população (não é a média dos % dos países)
% Idosos = DIVIDE ( [Idosos 65+], SUM ( fato_idosos[populacao_bm] ) )

% Idosos 2022 = CALCULATE ( [% Idosos], dim_ano[ano] = 2022 )

Ranking População 2030 =
IF (
    HASONEVALUE ( dim_pais[pais] ),
    RANKX ( ALL ( dim_pais[pais] ), [População 2030] )
)
```

Formatos: `População*` e `Idosos 65+` como número inteiro com separador de milhar (ou unidade de exibição *Bilhões* nos cartões); medidas `%` como Porcentagem com 1 casa decimal.

## 5. Páginas sugeridas

### Página 1 — Visão geral
- **Segmentação** por `dim_pais[continente]` (estilo botões) no topo.
- **4 cartões:** `População 2022`, `População 2030`, `Variação 2022–2030 %` e `% Idosos 2022`.
- **Gráfico de linhas:** eixo X `dim_ano[ano]` (tipo **Contínuo**), valores `População Histórica` e `População Prevista`. Deixe a linha prevista tracejada (Formatar → Linhas → Estilo da linha).
- **Mapa coroplético:** local `dim_pais[pais]`, saturação de cor `Variação 2022–2030 %`. Em azul ficam os países que crescem; em vermelho, os que encolhem (cor divergente com centro em 0).

### Página 2 — Países
- **Tabela:** `dim_pais[pais]`, `dim_pais[continente]`, `População 2022`, `População 2030`, `Variação 2022–2030 %` e `Ranking População 2030`. Use formatação condicional (barras de dados) na variação.
- **Gráfico de barras:** os 10 países que mais crescem (filtro *N superior* = 10 por `Variação 2022–2030 %`) e outro com os 10 que mais encolhem (*N inferior*).
- **Gráfico de colunas empilhadas:** eixo `dim_ano[ano]`, valor `População`, legenda `dim_pais[continente]`.

### Página 3 — Envelhecimento
- **Gráfico de linhas:** eixo `dim_ano[ano]` (1970–2024), valor `% Idosos`, legenda `dim_pais[continente]`.
- **Gráfico de dispersão:** X `fato_crescimento_idosos[cresc_pop_pct]`, Y `fato_crescimento_idosos[var_pct_idosos_pp]`, detalhes `dim_pais[pais]`, legenda `dim_pais[continente]`. Ele mostra a conclusão do notebook 05: quem mais encolhe é quem mais envelhece.
- **Gráfico de barras:** os 10 países com maior `pct_idosos_2022`.

### Toques finais
- **Dica de ferramenta** no mapa e na dispersão, com `População 2022`, `População 2030` e `% Idosos 2022`.
- **Sincronizar segmentações** (Exibir → Sincronizar segmentações), para o filtro de continente valer em todas as páginas.
- Uma caixa de texto com a fonte: *World Population e Banco Mundial (SP.POP.65UP.TO); previsão: tendência linear entre os dois últimos pontos*.

## 6. Tema de cores

O arquivo [`tema_forecast.json`](tema_forecast.json) aplica a paleta do projeto a todo o relatório. Para usá-lo, vá em **Exibir → Temas → Procurar temas** e escolha o arquivo.

O que o tema define:

| Elemento | Cor | Uso |
|---|---|---|
| Fundo da página | `#f3f2ef` (cinza claro) | Os visuais em branco se destacam do fundo |
| Texto | `#0b0b0b` / `#52514e` | Títulos em preto, rótulos em cinza escuro |
| Série 1 | `#2a78d6` azul | |
| Série 2 | `#eb6834` laranja | |
| Série 3 | `#1baf7a` verde-água | |
| Série 4 | `#eda100` amarelo | |
| Série 5 | `#e87ba4` rosa | |
| Série 6 | `#008300` verde | |
| Divergente | vermelho `#c23a3a` → cinza `#f0efec` → azul `#1c5cab` | Mapa de variação: vermelho encolhe, cinza ≈ 0, azul cresce |
| Bom / neutro / ruim | `#0ca30c` / `#fab219` / `#d03b3b` | Só para indicadores (KPI), nunca para séries |

A ordem das séries foi escolhida para que cores vizinhas continuem distinguíveis por quem tem daltonismo. Por isso, use-as na ordem, sem pular cores.

### Cores fixas por continente

O Power BI distribui as cores pela ordem em que as categorias aparecem. Assim, se um filtro tirar um continente, os outros mudam de cor. Para cada continente manter sempre a mesma cor, defina-as uma vez em cada visual com legenda de continente (**Formatar → Cores**):

| Continente | Cor |
|---|---|
| Africa | `#2a78d6` azul |
| Asia | `#eb6834` laranja |
| Europe | `#1baf7a` verde-água |
| North America | `#eda100` amarelo |
| Oceania | `#e87ba4` rosa |
| South America | `#008300` verde |

### Dicas de uso
- **Histórico × previsão:** use a mesma cor nas duas linhas (azul), com a da previsão **tracejada**. A legenda e o traço já mostram a diferença.
- **Mapa:** em **Formatar → Cores de preenchimento → fx**, escolha *Gradiente*, ative *Adicionar uma cor intermediária* e defina o centro como **0**. Assim, o zero fica cinza, os países que crescem ficam azuis e os que encolhem, vermelhos.
- **Dispersão por continente:** com 6 cores no mesmo gráfico, algumas ficam parecidas (por exemplo, amarelo e laranja). Ative os **rótulos de categoria** nos pontos de destaque e deixe a segmentação de continente à mão, para isolar um grupo.
- **Rótulos:** o amarelo, o rosa e o verde-água têm pouco contraste com o fundo branco. Em barras com essas cores, mostre os valores como rótulos de dados, e não só pela cor.
