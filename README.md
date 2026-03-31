# IS 566 Final Project: Adventure Works Data Platform

Well, my friends. We're here. The journey has been long and arduous, but we've learned a lot along the way. And to help you demonstrate (to me, a future interviewer, and most importantly, to _yourself_) that you have indeed learned _a ton_ here in this class, this final project will bring it all together in a single flow.

Over three weeks, you'll build a complete data platform — from source systems to analytics dashboards to AI agent access. Each week builds on the last, and by the end, you'll have a portfolio-ready project that demonstrates the full breadth of modern data engineering.

---

## What You're Building

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  PostgreSQL  │  │   MongoDB    │  │  REST API    │
│  (Sales)     │  │  (Chat Logs) │  │  (Analytics) │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       │     ┌───────────┘                 │
       │     │                             │
       ▼     ▼                             ▼
┌──────────────┐                 ┌──────────────┐
│   Python     │                 │   Prefect    │
│   Processor  │                 │   Flow       │
└──────┬───────┘                 └──────┬───────┘
       │                                │
       ▼                                ▼
┌─────────────────────────────────────────────────┐
│              Snowflake Warehouse                │
│  ┌─────────┐  ┌──────────┐  ┌────────────────┐  │
│  │  Raw    │→ │  dbt     │→ │  Dashboards &  │  │
│  │ Tables  │  │  Models  │  │  Agent Access  │  │
│  └─────────┘  └──────────┘  └────────────────┘  │
└─────────────────────────────────────────────────┘
```

---

## The Three Milestones

### [Milestone 1: Containerized ETL Pipeline](milestone-1-instructions.md)

You'll build a containerized Python ETL microservice that extracts data from PostgreSQL and MongoDB, stages it in Snowflake via internal stages, and loads it into raw tables using COPY INTO. Then you'll integrate this new data into the existing dbt project, creating staging models that combine your new sources with the Adventure Works warehouse data. You'll cap it off with a Snowsight dashboard showing sales trends.

**What you'll produce:** A working pipeline that runs in Docker, loads data into Snowflake, and transforms it with dbt.

### [Milestone 2: Orchestration, Data Quality, and Agent-Assisted Development](milestone-2-instructions.md)

You'll add a third data source (web analytics from a REST API) orchestrated by Prefect, implement comprehensive data quality testing, and set up dbt Cloud for CI/CD. There's a twist: you'll use an AI agent to help you build the Prefect flow, documenting the process in a PRD and agent interaction log.

**What you'll produce:** An orchestrated pipeline with automated testing and data quality checks.

### [Milestone 3: Agent Data Access and Portfolio Finalization](milestone-3-instructions.md)

You'll expose your dbt models to AI agents through a dbt MCP server, upgrade your documentation to be agent-friendly, and build a demo client that shows how agents interact with your data. Then you'll polish everything into a portfolio-ready project with architecture diagrams, technical decision documentation, and quantified metrics.

**What you'll produce:** A complete data platform with agent access, professional documentation, and a portfolio you'd be proud to show in an interview.

---

## What Your Portfolio Will Look Like

When you're done, your repository will demonstrate:

- **Multi-source data ingestion** from PostgreSQL, MongoDB, and REST APIs
- **Containerized ETL** with Docker and watermark-based incremental extraction
- **Cloud data warehousing** with Snowflake (staging, loading, transformation)
- **dbt transformation** with staging, intermediate, and analytical models
- **Workflow orchestration** with Prefect (scheduling, retries, error handling)
- **Data quality** with dbt tests and source freshness monitoring
- **CI/CD** with dbt Cloud (scheduled builds, pull request validation)
- **AI agent access** via the dbt MCP server
- **Production-grade documentation** including architecture diagrams, technical decisions, and quantified metrics

That's not a homework assignment. That's a data platform.

---

## Getting Started

1. Read this overview to understand the full picture
2. Open [Milestone 1 Instructions](milestone-1-instructions.md) and begin
3. Each milestone builds on the previous one — complete them in order

> [!TIP]
> The `compose.yml` and `.env.sample` files contain sections for all three milestones. Milestone 2 and 3 services are commented out — you'll uncomment them as you progress.

---

## Repository Structure

```
├── milestone-1-instructions.md        # Week 1: ETL Pipeline
├── milestone-2-instructions.md        # Week 2: Orchestration & Quality
├── milestone-3-instructions.md        # Week 3: Agent Access & Portfolio
│
├── compose.yml                        # Docker Compose (all milestones)
├── .env.sample                        # Environment variables template
│
├── dbt/                               # dbt project (models, tests, seeds)
├── processor/                         # Python ETL processor (Milestone 1)
├── prefect/                           # Prefect orchestration (Milestone 2)
├── templates/                         # Templates for M2 and M3 deliverables
│   ├── m2/                            # PRD, agent log, dbt Cloud guide, tests
│   └── m3/                            # Portfolio, reflection, MCP demo, docs guide
├── sql/                               # Reference SQL scripts
└── screenshots/                       # Screenshot evidence
```
