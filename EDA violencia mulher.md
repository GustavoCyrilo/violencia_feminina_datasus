## EDA — Violência contra a Mulher (DATASUS/SIM)

### Objetivo

Analisar os óbitos por violência contra mulheres no Brasil entre 2015 e 2022, usando dados do Sistema de Informações sobre Mortalidade (SIM) do DATASUS, acessados via biblioteca `pysus`.

---

### Contexto dos dados

- **Fonte:** DATASUS — Sistema de Informações sobre Mortalidade (SIM)
- **Acesso:** Servidor FTP público do governo via biblioteca `pysus`
- **Formato bruto:** arquivos `.dbc` por estado/ano, convertidos para DataFrame pandas
- [[CID-10-LISTA-PDF.pdf]]

---

### Decisões metodológicas

- **Recorte:** apenas registros do sexo feminino (`SEXO == 2`)
- **Causa:** códigos CID-10 de agressão: **X85 a Y09**
    - X60–X84 são lesões autoprovocadas (suicídio) — excluídos
    - X85–Y09 são agressões — incluídos
- **Período:** 2015 a 2022
- **Abrangência:** todos os 27 estados brasileiros
- **Total de registros filtrados:** 33.873 óbitos

---

### Código

#### Célula 1 — Imports

python

```python
import os
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
from pysus.online_data.SIM import download
```

---

#### Célula 2 — Códigos CID-10 de agressão

python

```python
codigos_x = ["X{}".format(i) for i in range(85, 100)]
codigos_y = ["Y0{}".format(i) for i in range(0, 10)]
codigos_agressao = codigos_x + codigos_y
```

**O que faz:**

- `range(85, 100)` gera números de 85 até 99 (o último é excluído em Python)
- `"X{}".format(i)` coloca o prefixo "X" na frente de cada número
- Resultado: lista `["X85", "X86", ..., "X99", "Y00", ..., "Y09"]`
- As duas listas são unidas com `+`

---

#### Célula 3 — Download, filtro e cache local

python

```python
if os.path.exists("violencia_feminina.parquet"):
    violencia_feminina = pd.read_parquet("violencia_feminina.parquet")
    print("Dados carregados do arquivo local!")
else:
    estados = ["AC", "AL", "AM", "AP", "BA", "CE", "DF", "ES", "GO",
               "MA", "MG", "MS", "MT", "PA", "PB", "PE", "PI", "PR",
               "RJ", "RN", "RO", "RR", "RS", "SC", "SE", "SP", "TO"]

    anos = list(range(2015, 2023))

    files = download(states=estados, years=anos, groups=["CID10"])

    dfs = []
    for file in files:
        df_temp = file.to_dataframe()
        df_filtrado = df_temp[
            (df_temp["SEXO"] == 2) &
            (df_temp["CAUSABAS"].str[:3].isin(codigos_agressao))
        ]
        dfs.append(df_filtrado)
        del df_temp  # libera memória imediatamente após filtrar

    violencia_feminina = pd.concat(dfs, ignore_index=True)
    violencia_feminina.to_parquet("violencia_feminina.parquet")
    print("Dados baixados e salvos!")

print(f"Total de registros: {violencia_feminina.shape[0]}")
print(f"Total de colunas: {violencia_feminina.shape[1]}")
```

**O que faz:**

- `os.path.exists()` verifica se o arquivo `.parquet` já existe em disco — evita baixar tudo de novo a cada sessão
- `download()` conecta ao FTP do DATASUS e baixa os arquivos por estado e ano
- O filtro é feito **durante** o carregamento (não depois) para economizar memória — `del df_temp` descarta o DataFrame completo imediatamente após filtrar
- `SEXO == 2` filtra apenas registros femininos (1 = masculino, 2 = feminino, conforme dicionário SIM)
- `str[:3]` pega os 3 primeiros caracteres do código CID-10 (ex: `"X850"` → `"X85"`) para comparar com a lista de códigos de agressão
- `isin()` verifica se o valor está dentro de uma lista — equivalente a vários `==` com `|`, mas muito mais limpo
- `pd.concat()` junta todos os DataFrames filtrados em um só
- `.to_parquet()` salva em formato comprimido — padrão em ciência de dados, muito mais rápido que CSV
### Dados populacionais — API IBGE (SIDRA)

#### Contexto

Para calcular a **taxa de mortalidade por 100.000 habitantes** é necessário cruzar os óbitos com a população de cada estado por ano. Usar números absolutos seria incorreto pois estados mais populosos teriam naturalmente mais óbitos.

**Fonte:** API pública do SIDRA/IBGE — Tabela 6579 (população residente estimada por UF)

**Fórmula da taxa:**

```
(óbitos / população) × 100.000
```

---

#### Código

##### Célula 4 — Buscar população por estado e ano na API do IBGE

python

```python
import requests

url = "https://servicodados.ibge.gov.br/api/v3/agregados/6579/periodos/2015|2016|2017|2018|2019|2020|2021|2022/variaveis/9324?localidades=N3[all]"

response = requests.get(url)
dados = response.json()
```

**O que faz:**

- `requests.get()` faz uma requisição HTTP para a API do IBGE
- A URL contém os parâmetros: tabela 6579, períodos de 2015 a 2022, variável 9324 (população), todas as UFs (`N3[all]`)
- `.json()` converte a resposta para um dicionário Python

---

##### Célula 5 — Transformar o JSON em DataFrame formato longo

python

```python
registros = []

for serie in dados[0]['resultados'][0]['series']:
    estado = serie['localidade']['nome']
    for ano, populacao in serie['serie'].items():
        registros.append({
            'ESTADO_NOME': estado,
            'ANO': int(ano),
            'POPULACAO': int(populacao)
        })

populacao_df = pd.DataFrame(registros)
print(populacao_df.head(10))
print(populacao_df.shape)
```

**O que faz:**

- O JSON da API vem aninhado em camadas — lista → resultados → series → cada estado
- O **loop externo** percorre cada estado (27 iterações)
- O **loop interno** percorre cada ano daquele estado com `.items()` — que retorna chave e valor ao mesmo tempo (ano e população)
- `registros.append()` adiciona um dicionário por combinação estado+ano
- `pd.DataFrame(registros)` converte a lista de dicionários em DataFrame
- Resultado: formato longo com uma linha por estado/ano — ideal para cruzar com os óbitos

**Por que formato longo?** O formato longo permite fazer `merge` direto combinando as colunas `ESTADO` e `ANO` das duas tabelas. No formato largo (anos como colunas) isso seria muito mais trabalhoso.

---

#### Estrutura do JSON retornado pela API

```
dados[0]
  └── resultados[0]
        └── series (lista de 27 estados)
              └── cada item:
                    ├── localidade → nome do estado
                    └── serie → {ano: população, ano: população, ...}
```