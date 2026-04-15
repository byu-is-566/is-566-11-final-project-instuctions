# Setting Up the dbt MCP Server in Docker Compose

> **Why this document exists:** The original Milestone 3 instructions described running the dbt MCP server locally. We've updated the approach to run it in Docker Compose instead, which is cleaner and consistent with how the rest of your services run. This guide walks you through the files you need to create. Credit to Logan McNatt for pioneering this approach.

---

## Overview

You'll create five new files in your `dbt/` directory and update your `compose.yml`. When you're done, running `docker compose up --build dbt-mcp` will:

1. Build a container with your dbt project and the MCP server
2. Run `dbt seed` and `dbt compile` to generate the `target/manifest.json` the MCP server needs
3. Start the MCP server on port 8000, accessible from your host machine

---

## Prerequisites

Before you begin:

- [x] Milestones 1 and 2 are complete (dbt models build, data flowing, tests passing)
- [x] Docker Desktop is running
- [x] Your `.env` file has valid Snowflake credentials (`SNOWFLAKE_USER`, `SNOWFLAKE_PASSWORD`, `SNOWFLAKE_ACCOUNT`, `SNOWFLAKE_WAREHOUSE`, `SNOWFLAKE_DATABASE`, `SNOWFLAKE_ROLE`)

---

## Step 1: Create `dbt/profiles.yml`

Create a new file at `dbt/profiles.yml` with the following content:

```yaml
adventure-ops:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: "{{ env_var('SNOWFLAKE_ROLE') }}"
      database: "{{ env_var('SNOWFLAKE_DATABASE') }}"
      warehouse: "{{ env_var('SNOWFLAKE_WAREHOUSE') }}"
      schema: dbt_dev
      threads: 4
```

**What's happening here:** This file tells dbt how to connect to Snowflake, but instead of hardcoding your credentials, it uses dbt's `env_var()` Jinja function to read them from environment variables at runtime. This is the same pattern your processor and Prefect services use — credentials live in your `.env` file, not in code. This file is safe to commit because it contains no secrets.

> [!NOTE]
> **Impact on local `dbt build`:** If you run `dbt build` from the `dbt/` directory, dbt will find this `profiles.yml` and try to use it. For that to work, your `SNOWFLAKE_*` environment variables must be set. You can either:
> - Continue using your `~/.dbt/profiles.yml` by running `dbt build --profiles-dir ~/.dbt`
> - Or export your env vars first: `set -a; source ../.env; set +a; dbt build`
> - **Windows (PowerShell):** `Get-Content ../.env | ForEach-Object { if ($_ -match '^\s*([^#=\s]+)\s*=\s*(.*)$') { Set-Item "env:$($matches[1])" $matches[2] } }; dbt build`

---

## Step 2: Create `dbt/pyproject.toml`

Create a new file at `dbt/pyproject.toml`:

```toml
[project]
name = "dbt-mcp-server"
version = "0.1.0"
requires-python = ">=3.12,<3.14"
dependencies = [
    "dbt-snowflake>=1.9.0",
    "dbt-mcp>=0.1.0",
]
```

Then generate the lock file:

```bash
cd dbt
uv sync
```

This creates a `uv.lock` file that the Docker build needs. You should see it install `dbt-snowflake` and `dbt-mcp` along with their dependencies.

> [!TIP]
> If you don't have `uv` installed, follow the [install guide](https://docs.astral.sh/uv/getting-started/installation/). You already used it in earlier milestones, so it should be available.

---

## Step 3: Create `dbt/start_mcp.py`

Create a new file at `dbt/start_mcp.py`:

```python
"""Wrapper to start dbt-mcp with configurable host binding.

The upstream dbt-mcp hardcodes host=127.0.0.1 in the FastMCP constructor,
which prevents Docker port mapping from working. This wrapper overrides
server.settings.host after creation so FASTMCP_HOST is respected.
"""

import asyncio
import os

from dbt_mcp.config.config import load_config
from dbt_mcp.config.transport import validate_transport
from dbt_mcp.mcp.server import create_dbt_mcp


def main() -> None:
    config = load_config()
    server = asyncio.run(create_dbt_mcp(config))

    host = os.environ.get("FASTMCP_HOST")
    if host:
        server.settings.host = host

    transport = validate_transport(os.environ.get("MCP_TRANSPORT", "stdio"))
    server.run(transport=transport)


if __name__ == "__main__":
    main()
```

**Why this file is necessary:** The `dbt-mcp` package hardcodes `host=127.0.0.1` when it starts its web server. Inside a Docker container, `127.0.0.1` means "only listen on the container's own loopback interface," which makes the server invisible to the outside world — even with Docker's `-p 8000:8000` port mapping. This wrapper patches the host to `0.0.0.0` (listen on all interfaces) so the port mapping works and you can reach the server from your host machine.

---

## Step 4: Create `dbt/entrypoint.sh`

Create a new file at `dbt/entrypoint.sh`:

```bash
#!/usr/bin/env bash
set -e

echo "==> Seeding dbt project..."
uv run dbt seed --profiles-dir . --project-dir .

echo "==> Compiling dbt project to generate target/manifest.json..."
uv run dbt compile --profiles-dir . --project-dir .

echo "==> Starting dbt MCP server on port 8000..."
exec uv run python start_mcp.py
```

Then make it executable:

```bash
chmod +x dbt/entrypoint.sh
```

> [!NOTE]
> **Windows users:** skip this step. `chmod` doesn't exist in PowerShell, and the `Dockerfile` already runs `RUN chmod +x entrypoint.sh` inside the Linux container — which is the only place the executable bit actually needs to be set.

**What each step does:**
- **`dbt seed`** loads any seed CSV files into Snowflake (these are reference data files in your `seeds/` directory)
- **`dbt compile`** parses your dbt project and generates `target/manifest.json`, which is the file the MCP server's lineage and model discovery tools depend on
- **`exec ... start_mcp.py`** starts the MCP server. The `exec` replaces the shell process with the Python process, which ensures Docker can properly send shutdown signals to it.

---

## Step 5: Create `dbt/Dockerfile`

Create a new file at `dbt/Dockerfile`:

```dockerfile
FROM python:3.12-slim
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

WORKDIR /app

# Install dependencies first for better layer caching
COPY pyproject.toml uv.lock ./
RUN uv sync --no-dev --frozen

# Copy dbt project files
COPY dbt_project.yml profiles.yml ./
COPY models-m1/ ./models-m1/
COPY models-m2/ ./models-m2/
COPY analyses/ ./analyses/
COPY tests/ ./tests/
COPY seeds/ ./seeds/
COPY entrypoint.sh start_mcp.py ./
RUN chmod +x entrypoint.sh

CMD ["./entrypoint.sh"]
```

> [!IMPORTANT]
> The `COPY models-m2/` line requires your Milestone 2 models directory to exist. If you haven't completed Milestone 2 yet, this build will fail. Complete M2 first, then come back here.

---

## Step 6: Update `compose.yml`

Open your `compose.yml`. Find the Milestone 3 comment block at the bottom (it looks like this):

```yaml
  # =========================================================================
  # Milestone 3: dbt MCP Server
  # =========================================================================
  # The dbt MCP server runs LOCALLY (not in Docker). Start it from the dbt/
  # directory so that lineage tools can find target/manifest.json:
  #
  #   cd dbt
  #   MCP_TRANSPORT=sse \
  #   ...
```

**Replace that entire block** with:

```yaml
  # =========================================================================
  # Milestone 3: dbt MCP Server
  # =========================================================================

  dbt-mcp:
    build: ./dbt
    container_name: dbt-mcp
    env_file:
      - .env
    ports:
      - "8000:8000"
    environment:
      - MCP_TRANSPORT=sse
      - DBT_PROJECT_DIR=.
      - DBT_PROFILES_DIR=.
      - FASTMCP_HOST=0.0.0.0
      - PYTHONUNBUFFERED=1
    restart: on-failure
```

**What each setting does:**
- **`env_file: .env`** passes your Snowflake credentials (and other env vars) into the container — this is how `profiles.yml`'s `env_var()` calls get their values
- **`FASTMCP_HOST=0.0.0.0`** tells our `start_mcp.py` wrapper to bind the server to all interfaces (see Step 3 for why)
- **`MCP_TRANSPORT=sse`** uses Server-Sent Events over HTTP, so the demo client can connect via `http://localhost:8000/sse`
- **`DBT_PROJECT_DIR` and `DBT_PROFILES_DIR`** tell dbt where to find project files inside the container

---

## Step 7: Build and Verify

Build and start the MCP server:

```bash
docker compose up --build dbt-mcp
```

You should see three phases in the output:

1. **Seeding:** `==> Seeding dbt project...` followed by seed completion messages
2. **Compiling:** `==> Compiling dbt project to generate target/manifest.json...` followed by model parsing
3. **Server start:** `==> Starting dbt MCP server on port 8000...` followed by:
   ```
   INFO [dbt_mcp.mcp.server] Registering dbt cli tools
   INFO [dbt_mcp.mcp.server] Registering dbt codegen tools
   INFO:     Uvicorn running on http://0.0.0.0:8000
   ```

From a **separate terminal**, verify the SSE endpoint:

```bash
curl -s http://localhost:8000/sse | head -3
```

> **Windows (PowerShell):** `curl.exe -s http://localhost:8000/sse | Select-Object -First 3`

You should see:

```
event: endpoint
data: /messages/?session_id=...
```

If you see that, your MCP server is running and ready for the demo client. Return to the Milestone 3 instructions and continue with Task 1.3.

---

## Troubleshooting

### "Connection refused" when curling the SSE endpoint

The container may still be building, seeding, or compiling. Check the logs:

```bash
docker compose logs dbt-mcp
```

Wait for the "Uvicorn running" message before testing.

### Build fails with "COPY failed: file not found ... models-m2"

Your Milestone 2 models directory doesn't exist yet. Complete Milestone 2 first — you need the `dbt/models-m2/` directory with your web analytics models.

### Snowflake authentication errors during seed/compile

Your `.env` file may have incorrect Snowflake credentials. Verify them by checking that your other services (processor, Prefect) can still connect to Snowflake. The dbt-mcp container reads from the same `.env` file.

### Port 8000 already in use

If you previously ran the MCP server locally, make sure it's stopped (Ctrl+C in its terminal). Only one process can use port 8000.

### Container exits immediately / restart loop

Check the logs with `docker compose logs dbt-mcp`. Common causes:
- Missing or invalid `profiles.yml` in the `dbt/` directory
- Snowflake warehouse is suspended with auto-resume disabled
- A dbt seed or compile error (check the log output for the specific error)

---

## Appendix: Running Locally (Fallback)

If Docker gives you trouble, you can run the MCP server locally as a fallback. This requires `uv` and `dbt-snowflake` on your PATH.

```bash
# Install dbt if not already available
uv tool install dbt-snowflake

# From the dbt directory
cd dbt

# Start the MCP server
MCP_TRANSPORT=sse \
DBT_PROJECT_DIR=. \
DBT_PROFILES_DIR=. \
uvx dbt-mcp
```

**Windows (PowerShell):**

```powershell
# Install dbt if not already available
uv tool install dbt-snowflake

# From the dbt directory
cd dbt

# Start the MCP server
$env:MCP_TRANSPORT = "sse"
$env:DBT_PROJECT_DIR = "."
$env:DBT_PROFILES_DIR = "."
uvx dbt-mcp
```

The server will listen on `http://localhost:8000/sse`, same as the Docker approach. Note that for the local approach, you'll need a `profiles.yml` in your `dbt/` directory (or at `~/.dbt/profiles.yml`) with working Snowflake credentials, and you should run `dbt compile` first to generate the manifest.
