# Pipeline de Dados — Arquitetura Medalhão (Olist)

Pipeline de engenharia de dados implementado no **Databricks**, seguindo a **Arquitetura Medalhão** (Bronze → Silver → Gold) com dados do e-commerce [Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce).

Projeto desenvolvido como atividade do **Visagio Rocket Lab 2026**.

---

## Arquitetura

```
Landing (CSVs + API)
       │
       ▼
   ┌────────┐     ┌────────┐     ┌────────┐
   │ Bronze │ ──▶ │ Silver │ ──▶ │  Gold  │
   └────────┘     └────────┘     └────────┘
   Dados brutos   Dados limpos   Data Marts
   + timestamp    + padronizados + KPIs
```

## Estrutura do Repositório

```
├── 00_Preparando_Ambiente.ipynb          # Criação do catálogo, schemas e volume
├── 01_Landing_To_Bronze.ipynb            # Ingestão dos CSVs e API do BCB
├── 02_Bronze_To_Silver.ipynb             # Limpeza, deduplicação e padronização
├── 03_Silver_To_Gold.ipynb               # Data Marts e rankings de negócio
├── pipeline_medalhao_olist.yaml          # Workflow Databricks (3 tasks sequenciais)
├── assets/
│   └── execucao_job.png                  # Screenshot da execução do Job
├── data/                                 # CSVs do dataset Olist (Kaggle)
│   ├── olist_customers_dataset.csv
│   ├── olist_geolocation_dataset.csv
│   ├── olist_order_items_dataset.csv
│   ├── olist_order_payments_dataset.csv
│   ├── olist_order_reviews_dataset.csv
│   ├── olist_orders_dataset.csv
│   ├── olist_products_dataset.csv
│   ├── olist_sellers_dataset.csv
│   └── product_category_name_translation.csv
└── README.md
```

## Notebooks

### 00 — Preparação do Ambiente
- Criação do catálogo `medalhao` no Unity Catalog
- Schemas: `bronze`, `silver`, `gold`
- Volume `landing` para armazenamento dos CSVs

### 01 — Camada Bronze (Ingestão)
- Ingestão dos 9 CSVs do Olist como tabelas Delta
- Coluna `timestamp_ingestion` adicionada em todas as tabelas
- Ingestão via API PTAX do Banco Central (cotação do dólar)
- Datas parametrizadas via widgets do Databricks

### 02 — Camada Silver (Transformação)
- **Deduplicação sênior** com Window Functions (`ROW_NUMBER` por `timestamp_ingestion`) em consumidores, produtos e vendedores
- Tradução de status e tipos de pagamento (inglês → português)
- Colunas derivadas de entrega: `tempo_entrega_dias`, `diferenca_entrega_dias`, `entrega_no_prazo`
- Tolerância a falhas com `try_to_timestamp` nas avaliações
- Calendário contínuo para cotação do dólar com `last(ignorenulls=True)`
- Tabela final `fat_pedido_total` com valores em BRL e USD (arredondados para 2 casas decimais)
- Otimização física com `OPTIMIZE` + `ZORDER` nas tabelas fato

### 03 — Camada Gold (Data Marts)

**Projeto 1 — Visão Comercial:**
- `gold.fat_vendas_comercial`: receita por ano/mês/categoria com métricas de volume e ticket médio
- Rankings: Top 5 produtos mais e menos vendidos

**Projeto 2 — Satisfação de Clientes:**
- `gold.fat_avaliacoes_clientes`: avaliações por categoria/vendedor/estado com percentual de satisfação
- Rankings: produto e vendedor mais/menos bem avaliados (ordenação composta por nota + volume)

## Workflow (Orquestração)

O pipeline é orquestrado via **Databricks Workflows** com 3 tasks sequenciais:

```
01_landing_to_bronze  →  02_bronze_to_silver  →  03_silver_to_gold
```

- **Trigger:** Diário às 13:00 (America/Sao_Paulo)
- **Tolerância a schema:** `overwriteSchema = true`
- **Modo de escrita Gold:** `overwrite`

## Tecnologias

- **Databricks** (Unity Catalog, Delta Lake, Workflows)
- **PySpark** (transformações e agregações)
- **Delta Lake** (OPTIMIZE + ZORDER)
- **API PTAX / Banco Central do Brasil** (cotação do dólar)
- **Python** (requests para ingestão de API)

## Dataset

[Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) — ~100k pedidos realizados entre 2016 e 2018 em marketplaces brasileiros.

## Como Executar

1. Importe os notebooks para o Databricks
2. Faça upload dos CSVs para o volume `/Volumes/medalhao/default/landing/`
3. Execute os notebooks na ordem (00 → 01 → 02 → 03) ou configure o Workflow usando o arquivo `.yaml`

---

Desenvolvido por **Mickael** — Visagio Rocket Lab 2026
