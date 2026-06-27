# ADR-002: Snowflake Data Warehouse — Environment & Ingestion Schema

**Date:** 2026-06-27  
**Status:** Accepted  
**Project:** RainyRoof Restaurant Website — GitMind Ingestion Layer  
**Deciders:** Harsh (Technical Lead)

---

## Context

Snowflake acts as the raw ingestion layer before data is promoted to Neo4j. Raw commits and Jira issue records land in Snowflake first, then a transform step builds the graph. This ADR documents environment config, table schema, and the ingest contract for the RainyRoof repo.

---

## Environment

### Account

| Parameter | Value |
|-----------|-------|
| Provider | Snowflake (free trial / student account) |
| Edition | Standard |
| Cloud | AWS `us-east-1` |
| Warehouse | `COMPUTE_WH` (X-Small, auto-suspend 60s) |
| Database | `GITMIND_DB` |
| Schema | `RAW` (ingestion), `PROCESSED` (post-transform) |

### Connection string (.env)

```
SNOWFLAKE_ACCOUNT=<orgname>-<accountname>
SNOWFLAKE_USER=<your_user>
SNOWFLAKE_PASSWORD=<your_password>
SNOWFLAKE_WAREHOUSE=COMPUTE_WH
SNOWFLAKE_DATABASE=GITMIND_DB
SNOWFLAKE_SCHEMA=RAW
```

### Python connector

```bash
pip install snowflake-connector-python==3.10.0
```

```python
import snowflake.connector

conn = snowflake.connector.connect(
    account=os.environ["SNOWFLAKE_ACCOUNT"],
    user=os.environ["SNOWFLAKE_USER"],
    password=os.environ["SNOWFLAKE_PASSWORD"],
    warehouse=os.environ["SNOWFLAKE_WAREHOUSE"],
    database=os.environ["SNOWFLAKE_DATABASE"],
    schema=os.environ["SNOWFLAKE_SCHEMA"],
)
```

---

## Schema — RAW Layer

### `RAW.COMMITS`

```sql
CREATE TABLE IF NOT EXISTS RAW.COMMITS (
    sha             VARCHAR(40)     NOT NULL PRIMARY KEY,
    message         VARCHAR(500),
    author_name     VARCHAR(200),
    author_email    VARCHAR(200),
    committed_at    TIMESTAMP_NTZ,
    branch          VARCHAR(100),
    repo_name       VARCHAR(200),
    ingested_at     TIMESTAMP_NTZ   DEFAULT CURRENT_TIMESTAMP()
);
```

Sample rows from RainyRoof:

| sha | message | author_name | branch |
|-----|---------|-------------|--------|
| `dd208dd` | Update README.md | FahimFBA | main |
| `e1c41b2` | Create CONTRIBUTING.md | FahimFBA | main |
| `2d1e29b` | responsivity 50% | FahimFBA | development |
| `4365881` | responsiveness 80% | FahimFBA | development |

---

### `RAW.JIRA_ISSUES`

```sql
CREATE TABLE IF NOT EXISTS RAW.JIRA_ISSUES (
    issue_id        VARCHAR(20)     NOT NULL PRIMARY KEY,
    title           VARCHAR(500),
    issue_type      VARCHAR(50),
    status          VARCHAR(50),
    priority        VARCHAR(20),
    project_key     VARCHAR(20),
    ingested_at     TIMESTAMP_NTZ   DEFAULT CURRENT_TIMESTAMP()
);
```

---

### `RAW.JIRA_COMMIT_LINKS`

Junction table — maps Jira issues to commits (many-to-many).

```sql
CREATE TABLE IF NOT EXISTS RAW.JIRA_COMMIT_LINKS (
    issue_id    VARCHAR(20)  NOT NULL,
    sha         VARCHAR(40)  NOT NULL,
    PRIMARY KEY (issue_id, sha),
    FOREIGN KEY (issue_id) REFERENCES RAW.JIRA_ISSUES(issue_id),
    FOREIGN KEY (sha)      REFERENCES RAW.COMMITS(sha)
);
```

---

## Schema — PROCESSED Layer

Denormalized view consumed by Neo4j ingest script.

```sql
CREATE OR REPLACE VIEW PROCESSED.COMMIT_ISSUE_MAP AS
SELECT
    c.sha,
    c.message,
    c.author_name,
    c.committed_at,
    c.branch,
    j.issue_id,
    j.title        AS issue_title,
    j.issue_type,
    j.priority
FROM RAW.COMMITS c
LEFT JOIN RAW.JIRA_COMMIT_LINKS l ON c.sha = l.sha
LEFT JOIN RAW.JIRA_ISSUES j ON l.issue_id = j.issue_id;
```

---

## Ingest Flow

```
GitHub API → fetch commits (paginated, GITMIND_MAX_PAGES)
         ↓
  UPSERT RAW.COMMITS   (ON CONFLICT DO NOTHING or MERGE)
         ↓
jira/issues.json → parse issues + commit arrays
         ↓
  UPSERT RAW.JIRA_ISSUES
  UPSERT RAW.JIRA_COMMIT_LINKS
         ↓
  Query PROCESSED.COMMIT_ISSUE_MAP
         ↓
  Neo4j MERGE nodes + relationships
```

---

## UPSERT pattern (idempotent)

```python
cursor.execute("""
    MERGE INTO RAW.COMMITS AS target
    USING (SELECT %s AS sha, %s AS message, %s AS author_name,
                  %s AS author_email, %s::TIMESTAMP_NTZ AS committed_at,
                  %s AS branch, %s AS repo_name) AS source
    ON target.sha = source.sha
    WHEN NOT MATCHED THEN INSERT (sha, message, author_name, author_email,
                                   committed_at, branch, repo_name)
                         VALUES  (source.sha, source.message, source.author_name,
                                   source.author_email, source.committed_at,
                                   source.branch, source.repo_name);
""", (sha, message, author_name, author_email, committed_at, branch, repo_name))
```

---

## Decisions

**Snowflake as staging, not primary store** — graph queries run on Neo4j. Snowflake gives audit trail, replay, and bulk analytics (e.g., commits per day, issue cycle time).

**LEFT JOIN in view** — commits with no Jira link still appear. GitMind fallback uses `PRECEDED_BY` chain for these.

**X-Small warehouse + auto-suspend** — RainyRoof has 72 commits, 28 issues. Full ingest finishes in seconds. No need for larger warehouse.

---

## Rejected Alternatives

| Option | Reason rejected |
|--------|----------------|
| PostgreSQL (Render) | Render free tier ephemeral disk; data lost on sleep |
| SQLite local | Not reachable from Render backend |
| Write directly to Neo4j (skip Snowflake) | Loses raw data layer; harder to replay/debug ingest bugs |

---

## Consequences

- `GITMIND_MAX_PAGES` env var controls GitHub pagination depth. Set to `3` for RainyRoof (72 commits, fits in 2 pages at 30/page).
- Snowflake free trial expires after 30 days. Plan migration to paid or swap to Neon/Supabase Postgres before demo.
- All writes go through `MERGE` — safe to re-run ingest without duplicates.
