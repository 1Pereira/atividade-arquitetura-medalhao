# CineData Analytics · Arquitetura Medalhão no Databricks

Pipeline de ETL que organiza um catálogo de filmes (base combinada TMDB/IMDb) em um Data Lakehouse no Databricks, seguindo a Arquitetura Medalhão (Bronze, Silver e Gold). O resultado é um Star Schema para consumo do time de BI e uma tabela de contexto para o time de IA.

Atividade 2 do Rocket Lab-SP.

## Arquitetura

```
Inputs   5 arquivos CSV no Volume inputs + cotação do dólar via API do Banco Central
   │
   ▼
Bronze   dado bruto em Delta, modo append, coluna ingestion_datetime
   │
   ▼
Silver   colunas em português, tipagem, deduplicação e regras de negócio
   │
   ▼
Gold     Star Schema (fato, dimensões e bridge tables) e tabela de contexto para IA
```

Todas as tabelas ficam no catálogo `cinedata_analytics`, nos schemas `bronze`, `silver` e `gold`.

## Notebooks

| Notebook | O que faz |
|----------|-----------|
| `notebooks/00_Organizacao_do_Ambiente.ipynb` | Cria o catálogo, o schema `bronze` e o Volume `inputs`, recebe o upload dos 5 arquivos CSV e valida se todos estão no Volume |
| `notebooks/01_Ingestao_Bronze.ipynb` | Lê os 5 CSVs do Volume e grava as tabelas Bronze; extrai a cotação do dólar da API PTAX do Banco Central, com as datas definidas por widgets |
| `notebooks/02_Bronze_to_Silver.ipynb` | Cria o schema `silver` e gera as 7 tabelas Silver com as regras de limpeza e de negócio |
| `notebooks/03_Silver_to_Gold.ipynb` | Cria o schema `gold`, monta o Star Schema e a tabela de contexto para IA e responde às perguntas do Desafio de Analytics |

## Tabelas por camada

**Bronze**
`tb_movies_info`, `tb_movies_financials`, `tb_movies_metrics`, `tb_credits_and_tags`, `tb_movies_reviews`, `tb_cotacao_dolar`

**Silver**
`tb_info_filmes`, `tb_financeiro_filmes`, `tb_metricas_engajamento`, `tb_avaliacoes_usuarios`, `tb_generos`, `tb_pessoas_empresas`, `tb_cotacao_dolar`

**Gold**
- Fato: `fact_movies_performance`
- Dimensões: `dim_movies`, `dim_genres`, `dim_people`, `dim_companies`, `dim_reviews`
- Bridge tables: `bridge_movie_genre`, `bridge_movie_person`, `bridge_movie_company`
- Contexto para IA: `tb_contexto_ia`

## Como executar

1. Importe a pasta `notebooks` no Workspace do Databricks.
2. No `00_Organizacao_do_Ambiente`, rode as seções 1 e 2, faça o upload dos 5 CSVs da pasta Inputs no Volume `cinedata_analytics.bronze.inputs` e rode a validação até aparecer `[OK]`.
3. Execute os notebooks `01`, `02` e `03` em ordem. No `01`, as datas da API podem ser ajustadas nos widgets `data_inicio` e `data_fim` (formato `MM-DD-AAAA`).

Os arquivos CSV não são versionados neste repositório.

## Orquestração

O pipeline é executado por um job do Databricks (Lakeflow Jobs) com quatro tarefas encadeadas, uma por notebook: `00_organizacao_ambiente` → `01_ingestao_bronze` → `02_bronze_to_silver` → `03_silver_to_gold`. Cada tarefa só começa quando a anterior termina com sucesso. A definição do job está em `pipelines/cinedata_analytics_pipeline.yaml`.

## Principais decisões

- **Bronze sem `inferSchema`**: todas as colunas são gravadas como texto, preservando a origem. A tipagem acontece na Silver.
- **Append na Bronze, deduplicação na Silver**: cada execução gera um novo lote; a Silver mantém apenas a versão mais recente de cada filme.
- **Conversão segura**: valores fora de contexto causados pelo Column Shift viram `NULL` sem interromper o pipeline. Isso inclui anos que caíram na coluna de popularidade em linhas deslocadas.
- **Cotação do dólar**: série diária contínua com Forward Fill para fins de semana e feriados. Os valores em BRL usam a cotação mais recente extraída da API.
- **Lucro e margem**: só são calculados quando orçamento e receita existem, evitando tratar receita ausente como prejuízo e divisões por zero.
- **Grão da fato**: um registro por filme lançado. Ids repetidos nas tabelas financeira e de métricas são consolidados antes dos joins, e as relações N:N (gêneros, pessoas e produtoras) ficam nas bridge tables, sem duplicar a fato.
- **Desafio de Analytics**: valores monetários exibidos no padrão brasileiro (`R$ 1.234.567,89`), apenas na exibição; as tabelas Gold mantêm os valores em `DECIMAL(18,2)`.
