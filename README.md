# Modern Data Engineering for AI Systems

A five-day course on building modern, production-ready data architectures for
AI systems: Delta Lakehouses, real-time streaming pipelines, vector databases
and RAG, automated data quality/governance/lineage, and end-to-end
architecture integration in a capstone project.

Built from the course's original slide decks and lab notebooks, adapted into
a readable [Quarto](https://quarto.org) website.

## Outline

**Day 1 — Modern Data Architectures**

1. Warehouses, lakes, and lakehouses
2. Decoupling compute from storage
3. ELT over ETL
4. Delta Lake and ACID transactions
5. Lab: build a local mini-lakehouse with Delta Lake

**Day 2 — Real-Time Data Pipelines**

1. Streaming and event-driven architectures
2. Kafka, windowing (tumbling/sliding/watermark), exactly-once delivery
3. Lab: real-time streaming pipeline with Kafka & Delta Lake

**Day 3 — Vector Databases and Advanced RAG Engineering**

1. Vector databases and embeddings
2. Chunking, hybrid search, reranking, RAG evaluation
3. Lab: vector databases and advanced RAG engineering
4. Exercise: build a RAG pipeline from scratch

**Day 4 — Data Quality, Governance, and Lineage**

1. Data quality: principles, validation, and contracts
2. Automated governance and enterprise policy
3. Lineage and data observability
4. Lab: data quality gates, Great Expectations, and OpenLineage on real retail data

**Day 5 — Architecture Integration and Final Project**

1. The unified architecture
2. The final project
3. Lab: architecture integration capstone

## Building the site locally

This is a [Quarto](https://quarto.org) website. Lab notebooks already contain
saved outputs from a real executed run, so rendering does not re-execute them
or require any local environment setup.

```
quarto render
quarto preview
```
