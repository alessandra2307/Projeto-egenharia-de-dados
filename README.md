# Projeto-egenharia-de-dados
# 🎵 SoundFlow Data

> **Disciplina:** Engenharia de Dados — Centro Universitário de Brasília (CEUB)
> **Professor:** Luis C. Cardoso
> **Equipe:** [Alessandra Gonçalves] · [Luiz Henrique] · [Rafael Mascarenhas]

Planejamento arquitetural de uma plataforma de engenharia de dados para um serviço de streaming musical. O projeto cobre o ciclo de vida completo dos dados — da ingestão ao consumo — aplicando conceitos de **Data Mesh**, **Domain-Driven Design**, **Arquitetura Lakehouse** e **Padrão Lambda**.

---

## 📋 Sumário

- [Sobre o Projeto](#sobre-o-projeto)
- [Domínios de Negócio](#domínios-de-negócio)
- [Arquitetura](#arquitetura)
- [Fontes e Classificação dos Dados](#fontes-e-classificação-dos-dados)
- [Stack Tecnológica](#stack-tecnológica)
- [Segurança e Governança](#segurança-e-governança)
- [Trade-offs](#trade-offs)
- [Estrutura do Repositório](#estrutura-do-repositório)
- [Parte 2 — Implementação](#parte-2--implementação)
- [Referências](#referências)

---

## Sobre o Projeto

O **SoundFlow Data** é uma plataforma de engenharia de dados inspirada em serviços como Spotify e Deezer. O problema central é processar dois tipos de dados com requisitos opostos:

- **Streaming em tempo real** — até 50.000 eventos/segundo (plays, skips, likes), com latência abaixo de 30 segundos para métricas ao vivo
- **Batch histórico de alta precisão** — cálculo exato de royalties, relatórios financeiros e enriquecimento de catálogo

A solução adota um **Lakehouse com Padrão Lambda**, unificando um único sistema de armazenamento para ambos os caminhos, eliminando a necessidade de manter dois sistemas separados sincronizados.

---

## Domínios de Negócio

O projeto segue os princípios de **baixo acoplamento e alta coesão** do Domain-Driven Design, organizados em cinco domínios com ownership claro sobre seus próprios dados:

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Catálogo  │  │  Usuários   │  │  Streaming  │  │  Financeiro │  │  Analytics  │
│             │  │             │  │             │  │             │  │             │
│ catalog-    │  │ user-       │  │ event-      │  │ billing-    │  │ data-wh-    │
│ ingestor    │  │ profile-svc │  │ collector   │  │ service     │  │ layer       │
│ catalog-    │  │ auth-       │  │ stream-     │  │ royalty-    │  │ ml-feature- │
│ enricher    │  │ service     │  │ processor   │  │ calculator  │  │ store       │
│ catalog-api │  │ subscription│  │ engagement- │  │ revenue-    │  │ dashboard-  │
│             │  │ service     │  │ aggregator  │  │ reporter    │  │ service     │
└─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘
                           │               │               │
                    ┌──────┴───────────────┴───────────────┴──────┐
                    │              Serviços Compartilhados          │
                    │  Airflow  │  Schema Registry  │  Great       │
                    │           │  (Contratos Avro) │  Expectations│
                    │  Grafana (Observabilidade)                    │
                    └───────────────────────────────────────────────┘
```

---

## Arquitetura

### Padrão Lambda + Arquitetura Medalhão

```
ORIGEM              INGESTÃO         ARMAZENAMENTO (LAKEHOUSE)     TRANSFORMAÇÃO    CONSUMO
                                     MinIO + Delta Lake
┌──────────┐     ┌──────────┐       ┌────────────────────┐
│ SDK      │────▶│  Kafka   │──────▶│   Bronze (raw)     │──Flink──▶ Metabase
│ (Events) │     │  (Avro)  │       │   imutável         │
└──────────┘     └──────────┘       ├────────────────────┤
                                    │   Silver (clean)   │──Spark──▶ API REST
┌──────────┐     ┌──────────┐       │   deduplicado      │           MLflow
│  Bancos  │────▶│ Airbyte  │──────▶├────────────────────┤
│  + APIs  │     │  (CDC)   │       │   Gold (analytics) │──dbt────▶ Metabase
└──────────┘     └──────────┘       │   pronto p/ consumo│           API REST
                                    └────────────────────┘
                      ▲─────────────────── Airflow (Orquestração Batch) ────────────────▶
```

| Camada | Responsabilidade | Tecnologia |
|--------|-----------------|------------|
| **Bronze** | Dado bruto, imutável, nunca deletado | Flink (streaming) / Airbyte (batch) |
| **Silver** | Limpeza: tipagem, deduplicação, nulos | Apache Spark |
| **Gold** | Modelos analíticos prontos p/ consumo | dbt (SQL versionado) |

---

## Fontes e Classificação dos Dados

| Fonte | Tipo | Formato | Volume | Periodicidade | Latência |
|-------|------|---------|--------|---------------|----------|
| SDK mobile/web | **Streaming** | Avro (Schema Registry) | 50k ev/s · ~2 bi/dia | Contínuo (tempo real) | < 500 ms |
| PostgreSQL (Usuários) | Operacional (CDC) | Registros relacionais | ~50 mi de perfis | CDC incremental diário | < 5 min |
| MySQL (Catálogo) | Operacional (CDC) | Registros relacionais | ~80 mi de faixas | CDC incremental diário | < 5 min |
| Stripe API (Financeiro) | Operacional (REST) | JSON | ~500k transações/mês | Pull incremental diário | < 30 min |
| DistroKid / TuneCore | Operacional (REST) | JSON / CSV | Metadados de faixas | Batch diário (02h00) | < 1 hora |

**Eventos de streaming capturados pelo SDK:**
- `play_event` — faixa iniciada (user_id, track_id, timestamp, device_type, session_id)
- `skip_event` — faixa pulada antes dos 30s (user_id, track_id, skip_at_second, timestamp)
- `like_event` — curtida em faixa ou álbum (user_id, entity_id, entity_type, timestamp)
- `search_event` — busca realizada (user_id, query_text, results_count, timestamp)
- `session_start` / `session_end` — abertura e fechamento de sessão

---

## Stack Tecnológica

### Ingestão

| Ferramenta | Função | Por que escolhemos |
|------------|--------|-------------------|
| **Apache Kafka** | Fila de streaming | 50k ev/s, retenção para replay via offset, replicação fator 3, desacoplamento produtor/consumidor |
| **Airbyte** | Extração batch | CDC incremental nativo para PG/MySQL/Stripe, conectores prontos, interface de monitoramento |

### Armazenamento

| Ferramenta | Função | Por que escolhemos |
|------------|--------|-------------------|
| **MinIO** | Object storage | 100% compatível com S3, roda local via Docker, zero vendor lock-in |
| **Delta Lake** | Formato de tabela | ACID sobre Parquet, time travel (auditoria de royalties), MERGE/upsert, Bronze imutável |

### Processamento

| Ferramenta | Função | Por que escolhemos |
|------------|--------|-------------------|
| **Apache Flink** | Streaming engine | Exactly-once semantics, janelas de tempo complexas, watermarks para eventos fora de ordem |
| **Apache Spark** | Motor batch | Escala para centenas de GB, integração nativa com Delta Lake, PySpark |
| **dbt** | Transformação SQL | SQL versionado no Git, testes automáticos embutidos, lineage automático Silver → Gold |

### Orquestração e Consumo

| Ferramenta | Função | Por que escolhemos |
|------------|--------|-------------------|
| **Apache Airflow** | Orquestração batch | DAGs Python, retry automático, operadores nativos Spark/dbt/Airbyte |
| **Metabase** | Dashboards | Open-source, SQL direto na camada Gold, sem conhecimento técnico avançado |
| **MLflow** | Ciclo de vida ML | Feature Store + tracking de experimentos + registro de modelos, integração com Spark |

---

## Segurança e Governança

### Segurança em múltiplas camadas

- **Autenticação:** JWT entre todos os serviços internos via `auth-service`
- **Criptografia em repouso:** AES-256 SSE-S3 em todos os buckets MinIO
- **Criptografia em trânsito:** TLS 1.2+ em toda comunicação
- **Credenciais:** Fernet key no Airflow — nenhuma credencial hardcoded em DAGs
- **Tokenização:** campos sensíveis (cartão, CPF) mascarados antes de entrar na Bronze

### Controle de acesso por camada

| Camada | Leitura | Escrita |
|--------|---------|---------|
| Bronze | Engenharia de dados | Flink e Airbyte (somente) — imutável, sem DELETE |
| Silver | Engenharia de dados + Analytics | Spark (jobs orquestrados) |
| Gold | Todos os times (produto, mkt, financeiro) | dbt |

### Governança

- **Schema Registry:** política `BACKWARD` — novos schemas não quebram consumidores existentes
- **Great Expectations:** validações automáticas entre cada camada (not_null, unique, integridade referencial)
- **dbt:** lineage automático de todas as tabelas Gold
- **Pipeline como código:** DAGs, modelos dbt, schemas Avro e configs do Airbyte versionados no Git

### Conformidade LGPD

- Dados pessoais identificáveis armazenados **apenas** no PostgreSQL (domínio Usuários)
- Pseudonimização: `user_id` (UUID) em todo o pipeline — sem PII no Lakehouse
- Direito ao esquecimento (art. 18): exclusão propagada via evento Kafka → anonimização em Bronze, Silver e Gold

---

## Trade-offs

| Decisão | Vantagem | Custo / Risco |
|---------|----------|---------------|
| Bronze imutável | Reversibilidade total — reprocessamento desde o dado original | Custo de armazenamento permanentemente elevado |
| Kafka desacoplado | SDK e Flink evoluem de forma independente | Schema Registry vira ponto de acoplamento; mudança mal gerenciada quebra todos os consumidores |
| Padrão Lambda | Cada caminho otimizado para seu requisito (latência vs. precisão) | Dois ecossistemas para operar (Kafka+Flink vs. Spark+Airflow) |
| Arquitetura Medalhão | Qualidade progressiva e rastreabilidade garantidas | Latência adicional — dado bruto só chega à Gold após Bronze e Silver |
| MinIO local | Zero custo, zero vendor lock-in, migração trivial para S3 | Sem replicação geográfica no ambiente de desenvolvimento |
| Flink exactly-once | Zero plays duplicados — contagem perfeita para royalties | Overhead de checkpointing de ~10-20% na latência |

> **Conclusão arquitetural:** Toda complexidade introduzida nesta arquitetura foi forçada por um requisito inegociável de negócio — não por preferência tecnológica.

---

## Estrutura do Repositório

```
soundflow-data/
│
├── docs/
│   ├── SoundFlow_Diagramas_Justificativas_Roteiro.pdf   # Documento completo (Parte 1)
│   ├── SoundFlow_Completo.docx                          # Versão editável do documento
│   └── SoundFlow_Lakehouse_Architecture.pdf             # Slides da apresentação
│
├── diagrams/
│   ├── dominios_servicos.md          # Diagrama de domínios (Mermaid)
│   └── arquitetura_ponta_a_ponta.md  # Fluxo ponta a ponta (Mermaid)
│
├── parte2/                           # (Implementação — em andamento)
│   ├── docker-compose.yml
│   ├── kafka/
│   ├── flink/
│   ├── spark/
│   ├── dbt/
│   ├── airflow/dags/
│   └── scripts/faker_events.py
│
└── README.md
```

---

## Parte 2 — Implementação

A Parte 2 consistirá na implementação prática do pipeline via **Docker Compose**, com dados sintéticos gerados pelo **Faker**. O plano está organizado em 6 fases:

- [x] **Fase 1 — Infraestrutura base:** Docker Compose com todos os serviços
- [ ] **Fase 2 — Ingestão:** Airbyte + conectores + script Python/Faker publicando no Kafka
- [ ] **Fase 3 — Processamento:** Job Flink (Kafka → Bronze) + Job Spark (Bronze → Silver) + Great Expectations
- [ ] **Fase 4 — Transformação & Consumo:** Modelos dbt (Silver → Gold) + Metabase + MLflow
- [ ] **Fase 5 — Qualidade & Governança:** Suite Great Expectations completa + Grafana
- [ ] **Fase 6 — Orquestração:** DAGs Airflow cobrindo o pipeline batch completo

> ⚠️ **Escopo consciente:** sem Alta Disponibilidade real (Airflow, Airbyte e MinIO em instância única). Objetivo: validar o fluxo, não garantir SLA de produção.

---

## Referências

- KLEPPMANN, Martin. *Designing Data-Intensive Applications*. O'Reilly Media, 2017.
- REIS, Joe; HOUSLEY, Matt. *Fundamentals of Data Engineering*. O'Reilly Media, 2022.
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Apache Flink Documentation](https://flink.apache.org/docs/)
- [Apache Spark Documentation](https://spark.apache.org/docs/latest/)
- [Delta Lake Documentation](https://docs.delta.io/)
- [dbt Documentation](https://docs.getdbt.com/)
- [Airbyte Documentation](https://docs.airbyte.com/)
- [Apache Airflow Documentation](https://airflow.apache.org/docs/)
- [MinIO Documentation](https://min.io/docs/minio/)
- [Great Expectations](https://greatexpectations.io/)
- [MLflow Documentation](https://mlflow.org/docs/latest/)
- BRASIL. Lei n.º 13.709/2018 — Lei Geral de Proteção de Dados Pessoais (LGPD).

---

<div align="center">
  <sub>Centro Universitário de Brasília — CEUB · Engenharia de Dados · 2026</sub>
</div>
