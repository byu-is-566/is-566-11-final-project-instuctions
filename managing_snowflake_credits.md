# Managing Your Snowflake Credits

You have a limited daily Snowflake credit budget for this project. This guide explains how that budget works, what burns through it fastest, and the habits that will keep you from running out at the worst possible moment.

Read this **once** before you start the hands-on work in Milestone 1. Skim it again whenever one of the milestone instructions points you here.

---

## How the Daily Quota Works

Each student has a **2-credit daily quota** on their personal Snowflake warehouse. 2 credits is roughly 2 hours of continuous XS-warehouse compute — plenty for normal development, but easy to blow through if something is left running unattended.

### When does the quota reset?

**Every day at approximately 6:00 PM Mountain Time** (or 5:00 PM in winter, before daylight saving).

Why that specific time? Snowflake resets daily resource monitors at **00:00 UTC**, which translates to 6:00 PM MDT (April–October) or 5:00 PM MST (November–March). This is controlled internally by Snowflake — it's not configurable, and it has nothing to do with the time your monitor was created.

**Practical implication:** if you stay up late hammering on dbt, then run a few more builds in the morning, those are all part of the **same** daily quota window. It won't reset until ~6 PM. That's why you can feel like you're "starting fresh" the next morning and still hit the limit quickly.

### What happens when you hit the limit?

When your warehouse reaches 100% of its daily quota, Snowflake **suspends the warehouse**. In-flight queries are allowed to finish, but new queries fail until the quota resets at the next 00:00 UTC. You'll see warehouse suspension errors in dbt, the Snowflake web UI, or whatever tool you're using.

If you hit the limit and you're genuinely stuck on an assignment deadline, reach out to me — I can manually reset your monitor in exceptional cases. But the better strategy is to avoid hitting the limit in the first place.

---

## What Consumes Credits?

Credits are consumed whenever your warehouse is **active** — executing a query, loading data, or otherwise doing work. Your warehouse is configured to auto-suspend after ~60 seconds of inactivity, so idle time is free. The things that burn credits fastest are:

1. **Continuous Docker Compose services.** The `processor` service runs on a timer (`PROCESSOR_INTERVAL_SEC`) and executes `COPY INTO` commands every cycle. The `web-analytics-flow` Prefect service does the same. If you leave these running overnight, every cycle wakes up your warehouse and costs credits.

2. **Repeated `dbt build` runs without `--select`.** A full `dbt build` rebuilds every model and runs every test. During development/troubleshooting, this adds up fast — especially if you're iterating every few minutes trying to get a model right.

3. **dbt Cloud scheduled jobs set to hourly.** This is the single worst offender we've seen. One student set their dbt Cloud schedule to "Every 1 hour" and woke up at 5 AM with zero credits left. Hourly runs in a 2 credit/day budget = bad math.

4. **Rebuilding dbt models on huge raw tables.** If you leave the generator running for hours, your `RAW_EXT` tables accumulate tens of thousands of rows. Every subsequent `dbt build` then has to transform all of those rows.

5. **The MCP `show` tool.** In Milestone 3, the dbt MCP server exposes a `show` tool that executes arbitrary SQL against Snowflake. Running it a few times for demos is fine; running it repeatedly against large result sets is not.

---

## Conservation Habits

Adopt these habits from day one and you'll almost certainly never hit your limit:

### 1. Stop Docker Compose when you're not actively working

```bash
docker compose down
```

This is the single most important habit. Close your laptop for the night? `docker compose down`. Taking a long break? `docker compose down`. Done for the day? `docker compose down`.

If you want to preserve your MongoDB/Postgres data between sessions, just use `docker compose down` (without `-v`). The volumes stick around and the services pick up where they left off when you bring them back up.

### 2. Use `dbt build --select` when iterating

Instead of rebuilding your entire project every time you tweak one model, scope your builds to just the model(s) you care about:

```bash
# Build just one model
dbt build --select stg_web_analytics

# Build that model and all downstream dependents
dbt build --select stg_web_analytics+

# Build that model and all upstream dependencies too
dbt build --select +stg_web_analytics+
```

Reserve full `dbt build` runs for when you've finished iterating and want a clean end-to-end verification.

### 3. Set dbt Cloud scheduled jobs to **daily**, not hourly

When you set up dbt Cloud in Milestone 2 (see `templates/m2/dbt_cloud_setup.md`), the schedule dropdown offers hourly options. **Do not pick them.** Set your scheduled job to run **once a day** at a specific time (e.g., "Every day at 8:00 AM"). This gives you daily production runs without draining your quota.

Also keep your **CI job** lean. It runs on every pull request, so if you open a lot of PRs while iterating, those runs add up. Use `dbt build` (not `dbt run && dbt test` separately) and consider using Slim CI with deferral if you learn how.

### 4. Reduce data volume when iterating on dbt models

If you've let the generator run for a while and your raw tables are huge, `dbt build` will burn credits every time. A quick reset:

```bash
# Tear down Docker AND wipe volumes (careful — this deletes your local data)
docker compose down -v

# Bring it back up briefly to generate a small amount of fresh data
docker compose up -d mongo postgres generator
# ... wait a minute or two ...
docker compose stop generator
```

Then in Snowflake, drop and recreate your `RAW_EXT` schema (or just truncate the tables) so you start with a small dataset. Your dbt builds will be dramatically faster and cheaper.

### 5. Make sure your warehouse auto-suspends

Your warehouse should auto-suspend after **60 seconds** of inactivity. Verify this in the Snowflake UI (Admin → Warehouses → your warehouse → Auto Suspend). If it's set higher, idle time between queries is costing you credits for no reason.

### 6. Turn off the processor's continuous loop

In your `.env` file, set `PROCESSOR_INTERVAL_SEC=0`. This makes the processor run once and exit, instead of running every 5 minutes forever. Use this mode whenever you don't specifically need continuous loading.

---

## Monitoring Your Usage

You can see your current credit consumption at any time. In a Snowflake worksheet, run:

```sql
SHOW RESOURCE MONITORS;
SELECT "name", "credit_quota", "used_credits", "remaining_credits", "frequency"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "name" = CURRENT_USER() || '_MONITOR';
```

Or use the Snowflake UI: **Admin → Resource Monitors**. Find the one named after your username (e.g., `BADGER_MONITOR`) and check the "Used Credits" column. If you're approaching your limit, stop running credit-heavy operations until the quota resets.

You can also check warehouse query history to see where your credits have been going:

```sql
SELECT query_type, warehouse_name, credits_used_cloud_services, start_time, execution_time/1000 AS exec_seconds
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE user_name = CURRENT_USER()
  AND start_time >= DATEADD(hour, -24, CURRENT_TIMESTAMP())
ORDER BY start_time DESC
LIMIT 50;
```

This shows your recent queries, how long they ran, and how much they cost. Useful for identifying specific offenders.

---

## What If I Run Out?

Your warehouse will suspend. `dbt build`, queries from the Snowflake UI, queries from your Python code — all will fail with a "warehouse suspended" error.

1. **Check the Resource Monitors UI** to confirm that's what happened.
2. **Wait for the next reset** (~6 PM MDT every day — see above).
3. **If you have a hard deadline**, reach out to me on Slack. I can manually reset your monitor, but this is a favor, not a right. Please have a good reason.

---

## FAQ

**Q: When exactly does my quota reset?**
Every day at **00:00 UTC**, which is ~6:00 PM MDT in summer (April–October) or ~5:00 PM MST in winter (November–March). Not midnight local time.

**Q: Can I change the reset time?**
No. Snowflake fixes it at 00:00 UTC for daily resource monitors. There's no configuration option.

**Q: I only ran `dbt build` twice today and I'm already at my limit. What gives?**
Two likely explanations: (1) you left Docker Compose running overnight and the processor was hitting Snowflake on a timer the whole time, and/or (2) your `dbt build` rebuilt every model every time instead of just the ones you were iterating on. Review the "Conservation Habits" section above.

**Q: Is it safe to just leave the Docker environment running in the background while I do other homework?**
No, unless you've set `PROCESSOR_INTERVAL_SEC=0` and stopped Prefect. The default configuration means the processor runs every 5 minutes and every run consumes credits.

**Q: My dbt Cloud schedule ran overnight and now I'm out. Can I just reschedule it?**
Yes — change it from hourly to daily (or disable it entirely while you're not needing production runs). Then wait for the next reset.

**Q: Does running a query from the Snowflake UI cost credits?**
Yes — any query you run against your warehouse consumes credits, whether it's from dbt, Python, the Snowflake UI, or anything else. Metadata operations (`SHOW TABLES`, `DESCRIBE`, etc.) are free.

**Q: Do idle warehouses cost credits?**
No. Once your warehouse auto-suspends (after ~60 seconds of inactivity), you're not paying for idle time. This is why it's safe to leave the Snowflake UI open between queries.
