# Projeto-egenharia-de-dados
# MotoAR — Plataforma de Qualidade do Ar para Motociclistas de Brasília

> Projeto de Engenharia de Dados — UniCEUB | Disciplina: Engenharia de Dados  
> Professor: Luis Carlos Cardoso | Brasília, 2026

**Equipe:**
- Alessandra Gonçalves
- Luiz Henrique 
- Rafael Mascarenhas

**Repositório:** [github.com/alessandra2307/Projeto-egenharia-de-dados](https://github.com/alessandra2307/Projeto-egenharia-de-dados)

---

## Sobre o Projeto

O **MotoAR** é uma plataforma de dados que coleta, processa e analisa informações de **qualidade do ar e condições meteorológicas em Brasília**, gerando recomendações práticas e personalizadas para motociclistas.

### O Problema

Brasília concentra mais de **500 mil motocicletas**, e seus condutores ficam expostos diretamente ao ar externo durante toda a jornada — sem qualquer filtração. Durante a estação seca (que pode durar de 4 a 6 meses), os índices de PM2.5 chegam a ultrapassar **5 vezes** os valores da estação chuvosa, superando frequentemente os limites recomendados pela OMS.

Apesar desse risco mensurável, **nenhuma ferramenta pública e acessível** integra dados de poluição atmosférica com recomendações de equipamento de proteção para motociclistas.

### A Solução

O MotoAR preenche essa lacuna com um pipeline robusto que:

- Coleta dados de múltiplas fontes (IQAir, INMET, Open-Meteo)
- Processa e transforma os dados em uma Arquitetura Medalhão com Delta Lake
- Treina um modelo preditivo (XGBoost) para gerar um **score de qualidade do ar para as próximas 6 horas**
- Entrega recomendações práticas ao motociclista: qual equipamento usar, alertas de queimada, inversão térmica noturna

---

## Fontes de Dados

| Fonte | Cobertura | Registros | Parâmetros | Frequência |
|---|---|---|---|---|
| INMET — CRAS Fercal | Jan–Dez 2025 | 8.760 | 14 | Horária |
| INMET — Escola | Jan–Dez 2025 | 8.760 | 8 | Horária |
| IQAir — Brasília | Mar 2026 | 9.853 | 5 | ~1 minuto |
| IQAir — Escola 115 | Mar 2026 | 8.766 | 5 | ~1 minuto |
| IQAir — UnB Gama | Mar 2026 | 9.053 | 5 | ~1 minuto |
| IQAir — Finatec | Mar 2026 | 8.629 | 5 | ~1 minuto |
| Open-Meteo | Futuro | — | 6 (forecast) | 6 horas |
| INMET 2024 | Jan–Dez 2024 | 8.784 | 14 | Horária |

**Total coletado:** 45.061 registros  
**Crescimento estimado:** ~17.520 novos registros/mês  
**Formato final de armazenamento:** Parquet (via Delta Lake)

---

## Arquitetura

O MotoAR utiliza a **Arquitetura Medalhão (Medallion Architecture)** com Delta Lake, organizada em três camadas progressivas de qualidade:

```
ORIGEM (APIs)
    │
    ▼
[INGESTÃO] ──── Python Requests + Schedule (batch + micro-batching a cada 5 min)
    │
    ▼
[BRONZE] ──────── MinIO + Delta Lake (dados brutos, imutáveis, versionados, ACID)
    │
    ▼
[SILVER] ──────── Apache Spark / PySpark + Great Expectations (limpeza, deduplicação, validação)
    │
    ▼
[GOLD] ─────────── dbt (86 features: temporais cíclicas, lags, rolling windows, indicadores)
    │
    ▼
[MODELO ML] ───── XGBoost + MLflow (Score MotoAR 0–100 + classe de risco)
    │
    ▼
[ENTREGA] ──────── Flask API + App MotoAR + Metabase + Push Notifications
```

**Por que Medalhão e não Lambda/Kappa?**
- **Lambda descartada:** implica duplicidade de código (batch + streaming) sem ganho de latência justificável
- **Kappa descartada:** o projeto não tem requisitos de streaming real em milissegundos; micro-batching a cada 5 min é suficiente
- **Data Mesh descartado:** excessivo para uma equipe de 3 pessoas

---

## Stack Tecnológico

| Etapa | Tecnologia | Justificativa |
|---|---|---|
| Ingestão | Python Requests + Schedule | Simplicidade, controle sobre polling de APIs REST |
| Armazenamento | MinIO + Delta Lake | S3-compatível local, ACID, time travel, zero vendor lock-in |
| Processamento | Apache Spark (PySpark) | Escalável, suporte nativo ao Delta Lake, joins e rolling windows |
| Transformação SQL | dbt | SQL versionado no Git, testes automáticos, lineage completo |
| Qualidade de Dados | Great Expectations | Validação automática, bloqueia dados inválidos antes de cada etapa |
| Orquestração | Apache Airflow | DAGs em Python, retry automático, maior ecossistema de operadores |
| Modelo ML | XGBoost + MLflow | Robusto para dados tabulares temporais; MLflow para rastreamento |
| Serviço | Flask API + Metabase | Endpoint REST /predict < 500ms; dashboards sem código |
| Monitoramento | Grafana + Prometheus | Alertas em tempo real, integração nativa com Airflow e Spark |

---

## Domínios de Negócio (DDD)

O projeto segue os princípios de **Domain-Driven Design**, dividido em 5 domínios com baixo acoplamento:

```
Ingestão → Processamento → Feature Engineering → Modelo ML → Entrega
```

| Domínio | Responsabilidade |
|---|---|
| **Ingestão** | Coleta de dados das APIs (IQAir, INMET, Open-Meteo), validação de schema |
| **Processamento** | Limpeza, deduplicação, outlier clipping, imputação de nulos |
| **Feature Engineering** | Geração das 86 features (lags, rolling windows, temporais cíclicas, indicadores) |
| **Modelo ML** | Treinamento XGBoost, geração do Score MotoAR 0–100, inferência via API |
| **Entrega** | Recomendações de equipamento, alertas críticos, dashboards Metabase |

**Serviços compartilhados:** Apache Airflow (orquestração) + Grafana/Prometheus (monitoramento)

---

## Resultados da Análise Exploratória (EDA)

Análise conduzida sobre **45.061 registros reais** (INMET 2025 + IQAir março 2026):

### PM2.5 — Estação CRAS Fercal (INMET 2025)

| Métrica | Valor |
|---|---|
| Média anual | 11,55 µg/m³ |
| Desvio padrão | 11,91 µg/m³ (alta variabilidade) |
| Pico máximo | **156,9 µg/m³** (outubro/2025 — queimadas) |
| P95 | 35,66 µg/m³ (classificação: Ruim — OMS) |
| P99 | 54,44 µg/m³ (classificação: Perigoso) |

### Principais Padrões Identificados

- **Sazonalidade:** PM2.5 na estação seca é **5,3× maior** do que na chuvosa (22,3 µg/m³ em setembro vs. 4,2 µg/m³ em maio)
- **Duplo pico diário:** 7h–9h (tráfego matutino, ~13 µg/m³) e 19h–21h (inversão térmica noturna, ~18 µg/m³ na seca)
- **Efeito da chuva:** dias com >5mm de precipitação têm PM2.5 **34% menor**
- **Heterogeneidade espacial:** sensor UnB Gama (periferia) tem AQI **70% maior** que Finatec (área verde/lagos)

---

## Roadmap — Parte 2 (Implementação)

| Semana | Entrega |
|---|---|
| Semana 1 | Infraestrutura Docker Compose (MinIO, Spark, PostgreSQL, Airflow, Grafana) |
| Semana 2 | Clientes de API + ingestão na camada Bronze do Delta Lake |
| Semana 2–3 | Spark jobs Bronze → Silver + validação Great Expectations |
| Semana 3 | Modelos dbt Silver → Gold com testes e lineage |
| Semana 4 | DAGs Airflow para pipeline batch diário + alertas Grafana |
| Semana 4 | Metabase dashboards + Flask API com inferência XGBoost |

### Critérios de Sucesso da Parte 2
- Pipeline batch rodando diariamente sem erros
- Dados Gold 100% testados e documentados (dbt)
- 5+ dashboards no Metabase utilizáveis
- Flask API retornando predições com **latência < 500ms**
- Grafana detectando anomalias em tempo real

---

## Riscos e Limitações

| Risco | Mitigação |
|---|---|
| Indisponibilidade das APIs externas | Cache da última leitura válida + retry automático no Airflow |
| Hardware limitado (< 8GB RAM) | Fallback para pandas em volumes pequenos no modo local |
| Cobertura geográfica parcial | Apenas 4 sensores IQAir; regiões satélite ainda sem cobertura adequada |
| Desvio do modelo em eventos atípicos | Retreinamento contínuo com novos dados; MLflow para rollback de versões |

---

## Referências

- [Delta Lake Documentation](https://docs.delta.io)
- [dbt Documentation](https://docs.getdbt.com)
- [Apache Airflow Documentation](https://airflow.apache.org/docs)
- [Apache Spark Documentation](https://spark.apache.org/docs/latest)
- [MinIO Documentation](https://min.io/docs/minio/container)
- [Great Expectations Documentation](https://docs.greatexpectations.io)
- [INMET — Dados de Estações Automáticas](https://portal.inmet.gov.br)
- [IQAir API Documentation](https://api-docs.iqair.com)
- [Open-Meteo API Documentation](https://open-meteo.com/en/docs)
- OMS. *WHO Global Air Quality Guidelines*. Genebra: WHO, 2021.
- Chen, T.; Guestrin, C. *XGBoost: A Scalable Tree Boosting System*. ACM SIGKDD, 2016.
- Kleppmann, M. *Designing Data-Intensive Applications*. O'Reilly, 2017.
- Reis, J. et al. *Fundamentals of Data Engineering*. O'Reilly, 2022.

---

*Projeto desenvolvido para a disciplina de Engenharia de Dados — UniCEUB, Brasília, 2026.*
