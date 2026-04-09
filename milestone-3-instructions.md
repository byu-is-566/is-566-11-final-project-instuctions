# Final Project // Milestone 3: Agent Data Access Layer and Portfolio Finalization

Over the past two weeks, you've built something substantial. You have a working data pipeline that extracts from multiple sources, loads into Snowflake, transforms with dbt, passes data quality tests, and serves analytics through dashboards. That's real data engineering.

This week, we're going to do two things. First, we'll make your data accessible to AI agents through a dbt MCP (Model Context Protocol) server. Second, we'll polish your entire project into something you'd be proud to show a hiring manager.

By the end of Milestone 3, you'll have:

- A dbt MCP server running in Docker Compose, exposing your data models to AI agents
- Agent-friendly documentation across all your dbt models
- A Python demo script that connects to the MCP server and demonstrates agent interaction
- A thoughtful reflection on what it means to design data for agent consumers
- A portfolio-ready README with architecture diagrams, technical decisions, and quantified metrics

This is the capstone. Let's make it count.

---

## Before You Start

A few orienting notes:

1. **Milestones 1 and 2 must be working.** Your dbt models should build successfully, your data should be flowing, and your tests should be passing. If anything is broken, fix it first.

2. **Docker Desktop must be running.** We're adding a new service to your Docker Compose configuration.

3. **Python 3.9+ with a virtual environment.** You'll need this for the MCP demo client script.

4. **Your `.env` file must have valid Snowflake credentials.** The MCP server will use these to connect to your warehouse.

> [!IMPORTANT]
> This milestone has two major components. **Component A** (Tasks 1-4) is about setting up the MCP server and demonstrating agent interaction. **Component B** (Task 5) is about finalizing your portfolio documentation. Both are required, and both matter for your grade.

---

## Task 1: Set Up the dbt MCP Server

The Model Context Protocol (MCP) is an open standard that lets AI agents discover and interact with external tools and data sources. The dbt MCP server exposes your dbt project (models, descriptions, compiled SQL) to any MCP-compatible agent. Think of it as giving an AI agent a structured way to ask questions about your data.

Why does this matter? Because data engineers are increasingly responsible for making data accessible not just to dashboards and reports, but to AI agents. An agent that can browse your dbt models, read your column descriptions, and compile SQL is a fundamentally different consumer than a human looking at a chart. Understanding this shift is valuable, both for your career and for this project's portfolio value.

> [!IMPORTANT]
> **Updated instructions:** The MCP server now runs in Docker Compose instead of locally. Follow the step-by-step setup guide in [**`m3_dbt_docker_setup.md`**](m3_dbt_docker_setup.md) (here in the instructions repo) to create the required files, then return here to continue with section 1.3 (verification).

### 1.1 - Prerequisites

Before setting up the MCP server, make sure you have:

1. **Docker Desktop running** — you've been using it since Milestone 1
2. **Milestones 1 and 2 complete** — your dbt models must build, and `dbt/models-m2/` must exist
3. **Valid Snowflake credentials in your `.env` file** — the MCP container reads from the same `.env` as your other services
4. **`uv` installed** — needed to generate a lock file and later for the demo client ([install guide](https://docs.astral.sh/uv/getting-started/installation/))

### 1.2 - Create Files and Start the MCP Server

Follow the **`m3_dbt_docker_setup.md`** guide to create five new files in your `dbt/` directory and update your `compose.yml`. The guide walks you through each file and explains what it does and why.

Once you've completed the setup guide, start the MCP server:

```bash
docker compose up --build dbt-mcp
```

You should see three phases: seeding, compiling, then the server starting:

```
==> Seeding dbt project...
==> Compiling dbt project to generate target/manifest.json...
==> Starting dbt MCP server on port 8000...
INFO [dbt_mcp.mcp.server] Registering dbt cli tools
INFO [dbt_mcp.mcp.server] Registering dbt codegen tools
INFO:     Uvicorn running on http://0.0.0.0:8000
```

**Leave this terminal running** (or add `-d` to run in the background). Open a new terminal for the next steps.

### 1.3 - Verify Server is Running

From a **separate terminal**, test the SSE endpoint:

```bash
curl -s http://localhost:8000/sse | head -3
```

You should see an SSE event stream like:

```
event: endpoint
data: /messages/?session_id=abc123...
```

If you get "Connection refused," the server hasn't finished starting. Wait a few seconds and try again.

### 1.4 - Troubleshooting

If things aren't working, check the container logs first:

```bash
docker compose logs dbt-mcp
```

Common issues:

- **"Connection refused"**: The container may still be building, seeding, or compiling. Wait for the "Uvicorn running" message in the logs before testing.
- **Build fails on `COPY models-m2/`**: Your Milestone 2 models directory doesn't exist yet. Complete M2 first.
- **Snowflake authentication errors during seed/compile**: Your `.env` file may have incorrect Snowflake credentials. Verify that your other services (processor, Prefect) can still connect.
- **Port 8000 already in use**: If you previously ran the MCP server locally, make sure it's stopped (Ctrl+C). Only one process can use port 8000.
- **Container exits immediately / restart loop**: Check the logs. Common causes include missing `profiles.yml`, suspended Snowflake warehouse, or dbt compile errors.

> [!TIP]
> If Docker gives you trouble, a local fallback approach is documented in the appendix of `m3_dbt_docker_setup.md`.

### Verification

- [ ] `curl -s http://localhost:8000/sse | head -3` returns an SSE event stream
- [ ] The server terminal shows no Snowflake authentication errors
- [ ] Server logs show "Registering dbt cli tools"

> [!IMPORTANT]
> 📷 Grab a screenshot showing the MCP server running in your terminal with the "Uvicorn running" message. Save this screenshot as `m3_task1.png` (or jpg) to the `screenshots` folder in the assignment repository.

---

## Task 2: Upgrade Your dbt Model Documentation

Here's an important insight: when an AI agent looks at your dbt models through the MCP server, the only way it understands what a model or column represents is through the `description` field in your YAML files. If your descriptions say "Order ID" for every ID column and "Staging model" for every staging model, the agent has almost nothing to work with.

Agent-friendly documentation is different from the minimal descriptions you might have written in earlier milestones. It explains business context, grain, relationships, and how columns connect across models. Improving these descriptions makes your data more accessible to agents AND to the humans who will read your project.

### 2.1 - Read the Agent-Friendly Documentation Guide

I've provided a guide at `templates/m3/AGENT_FRIENDLY_DOCS_GUIDE.md` that explains the principles of writing documentation that agents can actually use. Copy it into your `dbt/` directory so it lives alongside your models, then read through it before making changes to your YAML files:

```bash
cp templates/m3/AGENT_FRIENDLY_DOCS_GUIDE.md dbt/AGENT_FRIENDLY_DOCS_GUIDE.md
```

The key ideas:

1. **Every model description should explain what business entity it represents, what grain it has, and how it relates to other models.** Not just "staging model for orders."
2. **Every column description should explain the business meaning, not just echo the column name.** Not just "customer_id" for `customer_id`.
3. **Key columns (primary keys, foreign keys) should explicitly say they're used for joins** and name the target model.

### 2.2 - Upgrade Your Model Descriptions

Open your `dbt/models-m1/models.yml` (or whichever YAML file contains your model definitions). For every model in `staging/`, `intermediate/`, and `marts/` (if you have marts), update the description to follow the guide.

Here's a before/after example to calibrate your expectations:

```yaml
# BEFORE (minimal)
- name: stg_ecom__sales_orders
  description: "Staging model for sales orders"

# AFTER (agent-friendly)
- name: stg_ecom__sales_orders
  description: >
    Each row represents a single sales order from the e-commerce system.
    This staging model cleans and standardizes raw order data from the
    PostgreSQL source. Grain: one row per order. Join to
    stg_adventure_db__customers via customer_id to get customer details.
```

### 2.3 - Upgrade Your Column Descriptions

For every column in every model, add or improve the description. Focus especially on:

- **Primary keys**: Say it's a primary key and what table it uniquely identifies
- **Foreign keys**: Say what model this column joins to and on what field
- **Date/time columns**: Specify the timezone and what event the timestamp represents
- **Status or flag columns**: Explain the possible values and what they mean
- **Calculated columns**: Briefly explain the calculation logic

Example:

```yaml
columns:
  - name: order_id
    description: >
      Unique identifier for this sales order. Primary key.
      Source: PostgreSQL ecom.sales_orders.order_id.
      Join to int_sales_order_line_items on order_id to get line item details.
  - name: customer_id
    description: >
      Identifier of the customer who placed this order. Foreign key to
      stg_adventure_db__customers.customer_id. Use this join to access
      customer name, email, and demographic information.
  - name: order_date
    description: >
      Date and time the customer placed this order, in UTC.
      Used for time-based analysis like orders per month or daily revenue trends.
```

### 2.4 - Verify Completeness

Run a quick check to make sure you haven't missed any models or columns:

```bash
# Check for models without descriptions
grep -A1 "name:" dbt/models-m1/models.yml | grep -B1 'description: ""'

# Or, more simply, open the file and scan for any empty description fields
```

Every `description:` field should have meaningful content. No empty strings, no single-word descriptions, no descriptions that just repeat the column name.

> [!TIP]
> This is a place where an AI coding assistant can genuinely help. You can share your YAML file and the documentation guide with ChatGPT, Cursor, or Claude and ask it to suggest improved descriptions. Just make sure you review and customize the suggestions. The descriptions should be accurate to YOUR project, not generic.

### Verification

- [ ] Every model has a multi-sentence description (grain, business entity, key relationships)
- [ ] Every column has a description explaining business meaning
- [ ] No empty `description: ""` fields remain
- [ ] Key columns (PKs, FKs) explicitly mention join targets
- [ ] Descriptions use plain English, not SQL syntax

> [!IMPORTANT]
> 📷 Grab a screenshot showing one of your updated model YAML entries with full descriptions. Save this screenshot as `m3_task2.png` (or jpg) to the `screenshots` folder in the assignment repository.

---

## Task 3: Run the dbt MCP Demo Script

Now for the fun part. You're going to run a Python script that connects to your MCP server and demonstrates how an AI agent would interact with your data models. This is the kind of demo that makes a portfolio project come alive.

### 3.1 - Review the Demo Script

I've provided a starter script at `templates/m3/demo_client_starter.py`. Copy it to your working location and then open it to read through the code:

```bash
mkdir -p mcp
cp templates/m3/demo_client_starter.py mcp/demo_client.py
```

Open `mcp/demo_client.py` and read through the code. The script has six steps, each demonstrating a different MCP capability:

1. **Connect** to the MCP server via SSE
2. **List available tools** the server exposes
3. **Discover all dbt models** with their descriptions
4. **Get details** on a specific model (columns, tests, dependencies)
5. **Compile SQL** for a model (the fully resolved query dbt would execute)
6. **Explore model lineage** (upstream sources and downstream dependents)

The script is structured as a series of TODO blocks. You'll need to implement each one using the MCP Python SDK.

### 3.2 - Install Dependencies

Set up a virtual environment (if you don't already have one) and install the MCP client library:

```bash
cd mcp
cp ../templates/m3/mcp_pyproject.toml pyproject.toml
uv sync
```

> [!TIP]
> A `pyproject.toml` for the MCP client dependencies is provided as a template. Copy it into your `mcp/` directory and install:
> ```bash
> cp templates/m3/mcp_pyproject.toml mcp/pyproject.toml
> cd mcp && uv sync
> ```

The dependencies include `mcp` (the MCP Python client library), `httpx` (HTTP client for SSE transport), and `python-dotenv` (environment variable loading).

### 3.3 - Implement the Demo Steps

Open `demo_client.py` and fill in the TODO sections. The comments in each section explain what to do, and here are some additional hints:

**Connecting to the server (Step 1):**

```python
from mcp.client.sse import sse_client
from mcp.client.session import ClientSession

async with sse_client(MCP_SERVER_URL) as (read, write):
    async with ClientSession(read, write) as session:
        await session.initialize()
        # ... rest of your demo code goes here ...
```

**Listing tools (Step 2):**

```python
tools = await session.list_tools()
for tool in tools.tools:
    log(f"  - {tool.name}: {tool.description}")
```

**Calling a tool (Steps 3-6):**

```python
result = await session.call_tool("tool_name", {"param": "value"})
# Parse result.content to get the data
```

The exact tool names and parameters depend on what the dbt MCP server exposes. Use the tool list from Step 2 to discover what's available. Common tools include operations for listing models, getting model details, and compiling SQL.

> [!TIP]
> The MCP SDK uses async/await throughout. Make sure all your MCP calls are inside the `async with` blocks. If you're not familiar with async Python, that's okay. The pattern shown above is all you need. The key thing is that every `await` call must be inside an `async` function.

> [!IMPORTANT]
> The exact tool names may differ from what's shown in the comments. Use the output from Step 2 (listing tools) to discover the actual names, then update your Step 3-6 calls accordingly.

### 3.4 - Run the Demo

With the MCP server running and your script implemented:

```bash
cd mcp
uv run python demo_client.py
```

The script will print output to the console and save everything to `demo_output.log`.

### 3.5 - Review the Output

Open `mcp/demo_output.log` and verify you see:

1. A successful connection message
2. A list of available MCP tools
3. All your dbt models with their (improved) descriptions
4. Detailed information about at least one model (columns, tests, dependencies)
5. Compiled SQL for at least one model
6. Lineage information showing model relationships

If any step failed, the script should log the error and continue to the next step. Check the error messages for hints about what went wrong.

### 3.6 - Understand What Happened

Take a moment to think about what just happened. A Python script connected to a server that understands your dbt project. It discovered your models, read your documentation, compiled SQL, and traced lineage, all programmatically. This is exactly what an AI agent does when it needs to answer a question about your data.

The quality of the agent's answers depends directly on the quality of your model documentation (Task 2). If your descriptions are clear and complete, the agent can make informed decisions about which models to query and how to join them.

### Verification

- [ ] `python mcp/demo_client.py` runs without crashing
- [ ] `mcp/demo_output.log` contains output from all six steps
- [ ] Tool list, model list, model details, compiled SQL, and lineage are all present
- [ ] Error handling works (script continues past individual failures)

> [!IMPORTANT]
> 📷 Grab a screenshot showing the tail of your `demo_output.log` file. Save this screenshot as `m3_task3.png` (or jpg) to the `screenshots` folder in the assignment repository.

---

## Task 4: Write the Reflection on Agent Data Access

This reflection is not busywork. The questions it asks are exactly the kind of thing you'll discuss in data engineering interviews, especially as companies start thinking about how agents interact with their data platforms.

### 4.1 - Open the Reflection Template

Copy the reflection template to your `dbt/` directory and open it:

```bash
cp templates/m3/agent_access_reflection_template.md dbt/agent_access_reflection.md
```

Open `dbt/agent_access_reflection.md`. You'll find six guided prompts, each with context and hints to help you think deeply.

### 4.2 - Address Each Prompt

Write substantive responses (the entire reflection should be 500-800 words). Here's what makes a good response vs. a weak one:

**Weak**: "The MCP server worked well and the agent could see my models."

**Strong**: "The agent immediately understood my `stg_ecom__sales_orders` model because the description explicitly stated the grain (one row per order) and the join key to customers. However, it struggled with `int_sales_order_line_items` because I hadn't documented that the `discount_amount` column could be null for orders without promotions, which led the agent to write a query that produced unexpected results."

The six sections are:

1. **What Worked Well** - What aspects of your data models made agent interaction smooth?
2. **What Was Difficult** - Where did the agent struggle or need help?
3. **Documentation Quality** - How did improving descriptions change agent behavior?
4. **Production Considerations** - Security, cost, access control for real deployments
5. **Business Use Cases** - Specific problems agents could solve with your data
6. **The Bigger Picture** - How agent access changes the data engineering role

### 4.3 - Review and Polish

Read your reflection once more. Does it reference specific models and columns from YOUR project? Does it show genuine critical thinking? Could you discuss these ideas in a job interview?

### Verification

- [ ] All six sections have substantive responses
- [ ] Responses reference specific models, columns, or decisions from your project
- [ ] Total length is 500-800 words
- [ ] Responses show critical thinking, not placeholder text

---

## Task 5: Finalize Your Portfolio Documentation

This is the multi-part capstone. You're transforming your project from a class assignment into something you'd share with a hiring manager. This matters because most candidates show up to interviews talking vaguely about projects they've done. You'll show up with a polished, quantified, well-documented data platform.

### 5.1 - Restructure Your README into Portfolio Format

Replace your current assignment-focused `README.md` with a portfolio-ready version. Start by copying the provided template:

```bash
cp templates/m3/portfolio_readme_template.md README.md
```

Open `README.md` and fill in every `[TODO: ...]` placeholder with real content from your project. This is what a hiring manager will see first when they look at your repo, so make it count. Your README should include:

1. **Project title and one-line description** — what this is, in plain English
2. **Architecture diagram** — embed your diagram or link to the image from Task 5.2
3. **Problem statement** — the business context (Adventure Works needs visibility into web behavior)
4. **Tech stack with rationale** — not just a list of tools, but WHY each one
5. **Data flow** — end-to-end journey from source systems to analytics
6. **Setup and run instructions** — someone could clone your repo and get it running
7. **Key metrics** — quantified from actual queries (from Task 5.3)
8. **What I learned** — genuine reflection, not platitudes

> [!IMPORTANT]
> The tech stack section is where most students fall flat. Don't just list "Snowflake" and "dbt." Explain WHY. For example: "Snowflake was chosen for its separation of compute and storage, which prevents idle resource costs, and its native support for semi-structured data (JSON), which we use for MongoDB chat logs." That's the kind of rationale that shows understanding.

### 5.2 - Create an Architecture Diagram

Create a visual architecture diagram showing your complete data platform. Use any diagramming tool you like — Lucidchart, draw.io, Excalidraw, or even a whiteboard photo that's clean enough to read.

Your diagram should show:
- All data sources (PostgreSQL, MongoDB, REST API)
- The ETL processor and Snowflake stages
- dbt transformations (staging, intermediate)
- Prefect orchestration
- The MCP server and agent access layer

Export it as a PNG or SVG and include it in your repo (e.g., `screenshots/architecture.png`). Reference it in your README.

> [!TIP]
> Alternatively, you can embed a Mermaid diagram directly in your README — GitHub renders it natively. But a polished visual diagram often makes a stronger impression in a portfolio.

### 5.3 - (Optional) Document Technical Decisions

If you want to strengthen your portfolio further, create a `technical_decisions.md` documenting 3-5 key decisions you made. A template is available at `templates/m3/technical_decisions_template.md` with a worked example. Good decisions to document:

- Data warehouse choice (Snowflake vs. alternatives)
- Transformation framework (dbt vs. alternatives)
- Orchestration approach (Prefect vs. alternatives)
- Data loading strategy (internal stages, watermarks)

> [!TIP]
> In interviews, being able to say "I chose X because of Y, but I'd consider Z at larger scale" demonstrates mature engineering judgment. This is optional but powerful.

### 5.3 - Quantify Your Metrics

Go to Snowflake and run queries to get actual numbers. Don't estimate. Don't guess. Query your data and report what you find.

```sql
-- Example queries to calculate metrics
SELECT COUNT(*) as total_orders FROM your_schema.stg_ecom__sales_orders;
SELECT COUNT(*) as total_customers FROM your_schema.stg_adventure_db__customers;
SELECT COUNT(*) as total_chat_logs FROM your_schema.stg_real_time__chat_logs;

-- Pipeline latency (difference between source timestamp and warehouse load time)
-- Test coverage (count of dbt tests)
-- Availability (check dbt Cloud run history)
```

Put these numbers in your README's Key Metrics section and reference them in your technical decisions where relevant.

Metrics to quantify:

| Metric | How to Calculate |
|--------|-----------------|
| Records per cycle | Query raw table counts |
| Pipeline execution time | Time your Docker Compose processor run |
| dbt model count | `dbt ls --resource-type model \| wc -l` |
| dbt test count | `dbt ls --resource-type test \| wc -l` |
| Test pass rate | `dbt test` and count pass/fail |
| Data sources integrated | Count your sources in sources.yml |
| Models exposed via MCP | Count from demo script output |

### 5.4 - Final Polish

Before you call it done, review everything one more time:

- [ ] README reads smoothly from top to bottom — no placeholder text (`[FILL THIS IN]`, `[TODO]`)
- [ ] Architecture diagram renders correctly on GitHub
- [ ] Key metrics use specific numbers from actual queries, not estimates
- [ ] Setup instructions are complete enough that someone could clone and run your project
- [ ] `.env.sample` is up to date with all required variables
- [ ] `.gitignore` includes `.env`, `venv/`, `.venv/`, and any other sensitive/generated files
- [ ] No secrets in the repository — run `grep -r "eyJ\|password\|secret" . --include="*.py" --include="*.md" --include="*.yml"` to check
- [ ] All code blocks have syntax highlighting (` ```sql `, ` ```python `, ` ```bash `)

> [!TIP]
> **The hiring manager test:** Read your README as if you're a hiring manager with 5 minutes. Does the first screen tell them what this project does, what technologies you used, and why? Can they see the architecture at a glance? That's the bar. Most candidates show up to interviews talking vaguely about projects. You'll show up with a polished, quantified, well-documented data platform.

### Verification

- [ ] README has all sections filled in with real content
- [ ] Architecture diagram is present and covers all 3 milestones
- [ ] Key metrics are backed by actual Snowflake queries
- [ ] Setup instructions are complete (someone could clone and run)
- [ ] No placeholder text or TODO comments anywhere in documentation

> [!IMPORTANT]
> 📷 Grab a screenshot of your final README rendered on GitHub (push first, then view on GitHub). Save this screenshot as `m3_task5.png` (or jpg) to the `screenshots` folder in the assignment repository.

---

## Wrapping Up

Take a step back and look at what you've built across three weeks:

- **A multi-source ETL pipeline** that extracts from PostgreSQL, MongoDB, and a REST API
- **A cloud data warehouse** with staging, intermediate, and analytics layers
- **Data quality infrastructure** with automated tests, source freshness checks, and error handling
- **Orchestration and CI/CD** with Prefect and dbt Cloud
- **An agent data access layer** that exposes your models to AI through MCP
- **Portfolio documentation** that quantifies your work and demonstrates engineering judgment

That's a complete, production-style data platform. It demonstrates every major concept in modern data engineering, and it's yours.

### Commit and Push

```bash
git add .
git commit -m "Milestone 3: Agent data access layer and portfolio finalization"
git push origin main
```

### What to Submit

1. Your GitHub repository URL
2. Screenshots for Tasks 1, 2, 3, and 5 (saved in `screenshots/`)
3. The completed `agent_access_reflection.md`
4. The `mcp/demo_output.log` file (committed to your repo)

### Looking Ahead

Consider doing these (optional but valuable) things:

- **Record a 3-5 minute video walkthrough** of your project. Walk through the architecture, show the pipeline running, demonstrate the MCP demo. This is powerful in interviews.
- **Prepare to discuss your technical decisions.** If someone asks "Why Snowflake?", you should have a confident, nuanced answer.
- **Think about extensions.** What would you build next? Streaming ingestion? Real-time dashboards? A Slack bot that queries your data through MCP?

You deserve my CONGRATULATIONS. This has been a demanding three weeks, and you've built something that genuinely demonstrates data engineering competence. I hope you feel proud of what you've accomplished, because you should.
