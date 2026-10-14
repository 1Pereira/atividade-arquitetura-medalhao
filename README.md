# CineData Analytics · Pipeline de Dados End-to-End no Databricks

Pipeline de ETL que organiza um catálogo de filmes (base combinada TMDB/IMDb) em um Data Lakehouse no Databricks, seguindo a Arquitetura Medalhão (Bronze, Silver e Gold), com modelagem dimensional (Star Schema) para consumo do time de BI e orquestração por um Job agendado.

Atividade de Engenharia de Dados · RocketLab 2026.2.

## Arquitetura

```
Landing  5 arquivos CSV no Volume inputs + cotação do dólar via API do Banco Central
   │
   ▼
Bronze   dado bruto em Delta, modo append, coluna ingestion_datetime
   │
   ▼
Silver   colunas em português, tipagem, deduplicação e regras de negócio
   │
   ▼
Gold     Star Schema: fato, dimensões e bridge tables
```

Todas as tabelas ficam no catálogo `cinedata_analytics`, nos schemas `bronze`, `silver` e `gold`.

## Estrutura do repositório

```
├── notebooks/
│   ├── Landing_to_Bronze.ipynb
│   ├── Bronze_to_Silver.ipynb
│   └── Silver_to_Gold.ipynb
├── pipeline/
│   └── job.yaml
├── docs/
│   └── execucao_job.png
└── README.md
```

| Arquivo | O que faz |
|---------|-----------|
| `notebooks/Landing_to_Bronze.ipynb` | Cria o catálogo, o database `bronze` e o Volume `inputs`, valida os 5 CSVs, grava as tabelas Bronze e extrai a cotação do dólar da API PTAX, com as datas definidas por widgets |
| `notebooks/Bronze_to_Silver.ipynb` | Cria o database `silver` e gera as 7 tabelas Silver com as regras de limpeza e de negócio |
| `notebooks/Silver_to_Gold.ipynb` | Cria o database `gold`, monta o Star Schema e responde às perguntas do Desafio de Analytics |
| `pipeline/job.yaml` | Definição do Job de orquestração exportada do Databricks |
| `docs/execucao_job.png` | Execução de sucesso do Job, com as dependências entre as tarefas |

## Tabelas por camada

**Bronze**
`tb_movies_info`, `tb_movies_financials`, `tb_movies_metrics`, `tb_credits_and_tags`, `tb_movies_reviews`, `tb_cotacao_dolar`

**Silver**
`tb_info_filmes`, `tb_financeiro_filmes`, `tb_metricas_engajamento`, `tb_avaliacoes_usuarios`, `tb_generos`, `tb_pessoas_empresas`, `tb_cotacao_dolar`

**Gold**
- Fato: `fact_movies_performance`
- Dimensões: `dim_movies`, `dim_genres`, `dim_people`, `dim_companies`, `dim_reviews`
- Bridge tables: `bridge_movie_genre`, `bridge_movie_person`, `bridge_movie_company`

## Orquestração

O pipeline é executado pelo Job `cinedata_analytics_pipeline`, com três tarefas encadeadas:

```
to_Bronze  →  to_Silver  →  to_Gold
```

- Cada tarefa executa um notebook e só começa depois que a anterior termina com sucesso.
- Se algum CSV estiver faltando no Volume, a tarefa `to_Bronze` falha na validação e o Job para antes das camadas seguintes.
- **Agendamento:** de segunda a sexta, às 14h (horário de Brasília). A PTAX é divulgada em dias úteis no início da tarde, então cada execução já captura a cotação do dia.
- As datas da API usam o padrão dos widgets (últimos 7 dias a partir da execução), então não é preciso passar parâmetros ao Job.

## Como executar

1. Importe os notebooks da pasta `notebooks` no Workspace do Databricks.
2. No `Landing_to_Bronze`, rode as seções 1 e 2 e faça o upload dos CSVs da pasta Inputs no Volume `cinedata_analytics.bronze.inputs`.
3. Crie o Job a partir do `pipeline/job.yaml` (ajustando os caminhos dos notebooks) ou execute os três notebooks em ordem.

Os arquivos CSV não são versionados neste repositório.

## Principais decisões

- **Bronze sem `inferSchema`**: todas as colunas são gravadas como texto, preservando a origem. A tipagem acontece na Silver.
- **Append na Bronze, deduplicação na Silver**: cada execução gera um novo lote; a Silver mantém apenas a versão mais recente de cada filme.
- **Conversão segura**: valores fora de contexto causados pelo Column Shift viram `NULL` sem interromper o pipeline. Isso inclui anos que caíram na coluna de popularidade em linhas deslocadas.
- **Valores financeiros abreviados**: orçamentos como `200.0M` e `2.5K` são convertidos pela escala (K = mil, M = milhão, B = bilhão) em vez de virar `NULL`.
- **Separadores extras**: gêneros, pessoas e produtoras usam `;` e `|` além da vírgula; os três são padronizados antes do `split`.
- **Popularidade em linhas quebradas**: quando a origem divide a popularidade em duas colunas por aspas incorretas, o valor é reconstruído e as notas e votos voltam para a coluna correta.
- **Cotação do dólar**: série diária contínua com Forward Fill para fins de semana e feriados. Os valores em BRL usam a cotação mais recente extraída da API.
- **Lucro e margem**: só são calculados quando orçamento e receita existem, evitando tratar receita ausente como prejuízo e divisões por zero.
- **Grão da fato**: um registro por filme lançado. Ids repetidos nas tabelas financeira e de métricas são consolidados antes dos joins, e as relações N:N (gêneros, pessoas e produtoras) ficam nas bridge tables, sem duplicar a fato.
- **Desafio de Analytics**: valores monetários exibidos no padrão brasileiro (`R$ 1.234.567,89`), apenas na exibição; as tabelas Gold mantêm os valores em `DECIMAL(18,2)`.
