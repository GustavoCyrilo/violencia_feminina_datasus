## EDA — Violência contra a Mulher (DATASUS/SIM)

### Objetivo

Analisar os óbitos por violência contra mulheres no Brasil entre 2015 e 2021, cruzando dados de mortalidade com estimativas populacionais, para identificar padrões estaduais, tendências temporais e perfis de risco entre as unidades federativas.

Este documento funciona como um diário de desenvolvimento — registra não só as decisões finais, mas o processo de investigação, os erros encontrados e as correções aplicadas ao longo do projeto.

---

### Contexto dos dados

- **Óbitos:** Sistema de Informações sobre Mortalidade (SIM), DATASUS — Ministério da Saúde
- **População:** API do SIDRA/IBGE (Tabela 6579 — população residente estimada por UF)
- **Recorte:** sexo feminino, CID-10 X85 a Y09 (agressões), 2015 a 2021, 27 UFs
- [[CID-10-LISTA-PDF.pdf]]

---

## Etapa 1 — Tentativa inicial com `pysus`

A primeira abordagem foi acessar o SIM diretamente via biblioteca `pysus`, que se conecta ao FTP público do DATASUS.

### Código inicial

```python
import os
import pandas as pd
from pysus.online_data.SIM import download

codigos_x = ["X{}".format(i) for i in range(85, 100)]
codigos_y = ["Y0{}".format(i) for i in range(0, 10)]
codigos_agressao = codigos_x + codigos_y

estados = ["AC", "AL", "AM", ...]  # 27 UFs
anos = list(range(2015, 2022))

files = download(states=estados, years=anos, groups=["CID10"])

dfs = []
for file in files:
    df_temp = file.to_dataframe()
    df_filtrado = df_temp[
        (df_temp["SEXO"] == "2") &
        (df_temp["CAUSABAS"].str[:3].isin(codigos_agressao))
    ]
    dfs.append(df_filtrado)
    del df_temp

violencia_feminina = pd.concat(dfs, ignore_index=True)
violencia_feminina.to_parquet("violencia_feminina.parquet")
```

**Resultado inicial:** 33.873 registros, salvos em `.parquet` para evitar re-download a cada sessão.

---

## Etapa 2 — Problema descoberto: falhas silenciosas de download

Ao construir o gráfico de variação percentual (2015 vs 2021), surgiram valores matematicamente impossíveis — como Minas Gerais com uma variação de **+11.800%**.

### Investigação

Auditoria manual comparando o `.parquet` com o total nacional disponível no **TABNET/DATASUS**:

| Fonte | Total nacional (óbitos, 2015–2021) |
|---|---|
| Parquet (`pysus`) | 27.545 |
| TABNET (oficial) | 30.085 |

Diferença de ~8% — indicando falhas de download não sinalizadas por nenhum erro.

Auditando estado por estado, foram identificados casos graves:

| Estado | Ano | Óbitos no parquet | Óbitos reais (TABNET) |
|---|---|---|---|
| MG | 2015 | 2 | 415 |
| MT | 2016 | 2 | 104 |
| SC | 2016 | 2 | 107 |
| SP | 2016 | 2 | 506 |
| SP | 2021 | 1 | 342 |
| RJ | 2021 | 0 (arquivo vazio) | 4.054 |

Testes de re-download (`sim(state=..., year=...)`) para os mesmos estados/anos retornaram DataFrames vazios `(0, 0)` — confirmando que o problema era recorrente, não pontual.

**Decisão:** abandonar o `pysus` como fonte de dados e reconstruir a base inteira a partir do TABNET, a fonte oficial auditável.

---

## Etapa 3 — Migração para o TABNET

### Extração

Consulta manual no TABNET (`tabnet.datasus.gov.br`) com os filtros:
- Linha: Unidade da Federação
- Coluna: Ano do óbito
- Sexo: Feminino
- CID-10: X85 a Y09
- Período: 2015–2021

Exportado como `obitos_tabnet_2015_2021.csv`.

### Problemas de leitura do CSV

O arquivo exportado pelo TABNET trouxe vários obstáculos técnicos:

1. **Encoding** — o arquivo vinha em Latin-1, não UTF-8, causando `UnicodeDecodeError`
2. **Separador** — ponto e vírgula (`;`), não vírgula
3. **Linhas de cabeçalho** — 5 linhas descritivas antes da tabela de dados
4. **Linha de rodapé** — linha vazia ao final

**Solução adotada:** limpeza manual no LibreOffice Calc (remoção das linhas extras, padronização do cabeçalho) e exportação final em UTF-8 com separador vírgula — mais simples e confiável do que tentar contornar tudo via parâmetros do `pd.read_csv()`.

### Transformação dos dados

```python
df_tabnet = pd.read_csv("obitos_tabnet_2015_2021.csv").dropna()

# Extrai os 2 primeiros caracteres do código IBGE e mapeia para sigla
df_tabnet["ESTADO"] = df_tabnet["ESTADO"].str[:2].map(codigo_estado)

# Transforma formato largo (ano por coluna) em longo (uma linha por estado/ano)
obitos_agrupados = df_tabnet.melt(
    id_vars="ESTADO",
    var_name="ANO",
    value_name="OBITOS"
)

obitos_agrupados["ANO"] = obitos_agrupados["ANO"].astype(int)
obitos_agrupados["OBITOS"] = obitos_agrupados["OBITOS"].astype(int)
```

**Resultado:** 189 linhas (27 estados × 7 anos), auditadas e consistentes com o TABNET.

---

## Etapa 4 — Dados populacionais e correção da população feminina

### Fonte inicial

API do SIDRA/IBGE, Tabela 6579 (variável 9324 — população residente estimada), consultada por UF e ano:

```python
url = "https://servicodados.ibge.gov.br/api/v3/agregados/6579/periodos/2015|2016|2017|2018|2019|2020|2021/variaveis/9324?localidades=N3[all]"
response = requests.get(url)
dados = response.json()
```

### Problema identificado

A variável 9324 traz **população total** (todos os sexos), não população feminina. Isso subestimava sistematicamente a taxa por 100k em aproximadamente 2x.

### Investigação de alternativas

- Tentativa da variável **9326** (suposta população feminina) na Tabela 6579 → erro 500, variável inexistente nessa tabela
- Tabela **7358** (População, por sexo e idade) → tem a desagregação por sexo, mas cobre **apenas o ano de 2018** — insuficiente para a série 2015–2021

Nenhuma fonte do IBGE oferece população feminina por UF para o período completo via API.

### Decisão

Estimar a população feminina como **51% da população total**, percentual consistente com a distribuição demográfica brasileira segundo o IBGE:

```python
# Cria a estimativa da população feminina (51% do total)
tabela_unificada["POP_FEMININA"] = (tabela_unificada["POPULACAO"] * 0.51).astype(int)

tabela_unificada["TAXA_POR_100K"] = calcular_taxa_100k(
    tabela_unificada["OBITOS"],
    tabela_unificada["POP_FEMININA"]
)
```

A coluna `POPULACAO` original foi preservada para auditoria — `POP_FEMININA` é uma coluna derivada, não uma substituição.

**Impacto:** a correção é uniforme (mesmo fator para todos os estados/anos), então não altera rankings, variações percentuais nem os clusters do K-Means — apenas corrige a magnitude absoluta das taxas.

---

## Etapa 5 — Visualizações

### 1. Evolução nacional (linha)

Soma bruta de óbitos e população por ano antes de calcular a taxa — agregar taxas diretamente (ex: média simples de taxas anuais) seria matematicamente incorreto.

```python
evolucao_brasil = (
    tabela_unificada.groupby("ANO")[["OBITOS", "POPULACAO", "POP_FEMININA"]]
    .sum()
    .reset_index()
)
evolucao_brasil["TAXA_BRASIL"] = calcular_taxa_100k(
    evolucao_brasil["OBITOS"], evolucao_brasil["POP_FEMININA"]
)
```

Eixo Y com limites dinâmicos (`min() * 0.85`, `max() * 1.15`) para evitar valores fixos que quebrariam se os dados mudassem.

### 2. Ranking por estado (barras horizontais)

Mesma lógica de soma bruta antes da taxa, agrupando por `ESTADO` em vez de `ANO`. Barras horizontais escolhidas por comportar melhor os 27 rótulos de UF.

### 3. Variação percentual 2015 vs 2021 (barras divergentes)

```python
df_pivot = tabela_unificada.pivot(index="ESTADO", columns="ANO", values="TAXA_POR_100K")
df_pivot["VARIACAO_PERCENTUAL"] = ((df_pivot[2021] - df_pivot[2015]) / df_pivot[2015]) * 100
```

Eixo X normalizado com **limites simétricos** (`-limite, limite`) — decisão importante para não distorcer visualmente a percepção de magnitude entre aumentos e quedas.

**Achado principal:** 21 dos 27 estados (77,7%) reduziram a taxa no período; 6 estados pioraram, concentrados majoritariamente em Norte/Nordeste (AC +30,9%, CE +27,4%, BA +21,9%, AM +8,2%).

---

## Etapa 6 — Machine Learning: Clusterização K-Means

### Objetivo

Agrupar os 27 estados por perfil de mortalidade considerando três dimensões simultaneamente — taxa inicial, taxa final e taxa média histórica — em vez de analisar cada uma isoladamente.

### Features escolhidas

| Feature | Descrição |
|---|---|
| `TAXA_2015` | Taxa no início do período |
| `TAXA_2021` | Taxa no fim do período |
| `TAXA_ESTADO` | Taxa média histórica (2015–2021) |

**Decisão de design:** a variável `VARIACAO_PERCENTUAL` foi deliberadamente excluída das features por ser matematicamente derivada de `TAXA_2015` e `TAXA_2021` — incluí-la geraria redundância (multicolinearidade), dando peso duplicado à dimensão de mudança temporal no cálculo de distância do K-Means.

### Normalização

```python
scaler = StandardScaler()
features_scaled = scaler.fit_transform(features)
```

`StandardScaler` escolhido em vez de `MinMaxScaler` por lidar melhor com outliers — Roraima possui taxa muito acima dos demais estados.

### Escolha de K — Método do Cotovelo

```python
inertias = []
for k in range(1, 11):
    kmeans = KMeans(n_clusters=k, n_init=10)
    kmeans.fit(features_scaled)
    inertias.append(kmeans.inertia_)
```

O parâmetro `n_init=10` foi essencial — sem ele, a curva de inércia apresentava uma anomalia (subida entre K=4 e K=5) causada pela aleatoriedade da inicialização dos centroides. Com `n_init=10`, a curva ficou suave e o cotovelo claro em **K=5**.

### Modelo final e interpretação

```python
kmeans_final = KMeans(n_clusters=5, n_init=10, random_state=42).fit(features_scaled)
df_ml["cluster"] = kmeans_final.labels_
```

Os 5 clusters foram nomeados com base no perfil médio de cada grupo:

| Cluster | Estados | Perfil |
|---|---|---|
| Alto Risco Crescente | AC, AM, BA, CE | Taxas altas e subindo |
| Alto Risco em Queda | ES, GO, MT, PA, RO, TO | Taxas altas em 2015, caíram bastante |
| Caso Crítico | RR | Outlier extremo — taxa muito acima de qualquer outro estado |
| Baixo Risco | DF, MG, SC, SP | Menores taxas do país |
| Perfil Intermediário | Demais 12 estados | Comportamento médio, sem extremos |

**Achado relevante:** todos os clusters, exceto "Alto Risco Crescente", tiveram queda ou estabilidade na taxa — reforçando que os 4 estados desse grupo (AC, AM, BA, CE) merecem atenção prioritária em políticas públicas.

---

## Etapa 7 — Decisões descartadas: Regressão Linear

Cogitou-se aplicar regressão linear para projetar a taxa de mortalidade (Brasil e Roraima especificamente) para 2022–2023.

**Motivo do descarte:** a série temporal possui apenas 7 pontos anuais (2015–2021), volume insuficiente para produzir projeções estatisticamente confiáveis. Um modelo treinado com tão poucos dados teria alta variância e baixo poder preditivo real, ainda que tecnicamente executável.

**Decisão:** priorizar a qualidade e honestidade metodológica do portfólio em vez de incluir um modelo preditivo pouco robusto. Séries temporais mais longas (mais anos de dados, ou granularidade mensal) seriam necessárias para essa análise ter valor real — fica como direção para um projeto futuro com dados mais extensos.
