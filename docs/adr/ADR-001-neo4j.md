# ADR-001: Neo4j Graph Database — Environment & Schema Design

**Date:** 2026-06-27  
**Status:** Accepted  
**Project:** RainyRoof Restaurant Website — GitMind Knowledge Graph Layer  
**Deciders:** Harsh (Technical Lead)

---

## Context

GitMind ingests commit history and Jira-linked issues from this repo into a causal knowledge graph. Neo4j is the graph store. This ADR documents the environment setup, node/relationship schema, and constraints specific to the RainyRoof repo.

---

## Environment

### Instance

| Parameter | Value |
|-----------|-------|
| Provider | Neo4j Aura (Free Tier) |
| Edition | AuraDB Free |
| Neo4j Version | 5.x |
| Region | GCP `us-east1` (default Aura free) |
| Max nodes | 200,000 |
| Max relationships | 400,000 |

### Connection

```
NEO4J_URI=neo4j+s://<your-aura-instance-id>.databases.neo4j.io
NEO4J_USER=neo4j
NEO4J_PASSWORD=<aura-generated-password>
```

Store in `.env`. Never commit.

### Python driver

```bash
pip install neo4j==5.19.0
```

```python
from neo4j import GraphDatabase

driver = GraphDatabase.driver(
    os.environ["NEO4J_URI"],
    auth=(os.environ["NEO4J_USER"], os.environ["NEO4J_PASSWORD"])
)
```

---

## Schema

### Node Types

#### `Commit`
```cypher
CREATE CONSTRAINT commit_sha IF NOT EXISTS
FOR (c:Commit) REQUIRE c.sha IS UNIQUE;
```

| Property | Type | Example |
|----------|------|---------|
| `sha` | string | `"dd208dd"` |
| `message` | string | `"Update README.md"` |
| `author` | string | `"FahimFBA"` |
| `timestamp` | datetime | `2022-05-14T...` |
| `branch` | string | `"main"` |

#### `JiraIssue`
```cypher
CREATE CONSTRAINT jira_id IF NOT EXISTS
FOR (j:JiraIssue) REQUIRE j.id IS UNIQUE;
```

| Property | Type | Example |
|----------|------|---------|
| `id` | string | `"RRW-011"` |
| `title` | string | `"Responsiveness implementation"` |
| `type` | string | `"Story"` |
| `status` | string | `"Done"` |
| `priority` | string | `"High"` |

#### `Developer`
```cypher
CREATE CONSTRAINT dev_name IF NOT EXISTS
FOR (d:Developer) REQUIRE d.name IS UNIQUE;
```

#### `File`
```cypher
CREATE CONSTRAINT file_path IF NOT EXISTS
FOR (f:File) REQUIRE f.path IS UNIQUE;
```

#### `Branch`
```cypher
(:Branch {name: "main"})
(:Branch {name: "development"})
```

---

### Relationship Types

```
(Commit)-[:PRECEDED_BY]->(Commit)          // commit chain, ordered by timestamp
(Commit)-[:AUTHORED_BY]->(Developer)
(Commit)-[:MODIFIES]->(File)
(JiraIssue)-[:IMPLEMENTED_BY]->(Commit)    // from jira/issues.json commit array
(Commit)-[:BELONGS_TO]->(Branch)
(JiraIssue)-[:DEPENDS_ON]->(JiraIssue)     // optional, for future sprint planning
```

---

### Seed Cypher — Jira → Commit linkage (RainyRoof)

```cypher
// Example: RRW-011 linked to responsiveness commits
MERGE (j:JiraIssue {id: "RRW-011", title: "Responsiveness implementation (mobile/tablet)", type: "Story", status: "Done", priority: "High"})
MERGE (c1:Commit {sha: "2d1e29b"}) SET c1.message = "responsivity 50%"
MERGE (c2:Commit {sha: "4365881"}) SET c2.message = "responsiveness 80%"
MERGE (c3:Commit {sha: "f84045f"}) SET c3.message = "Merge pull request #6 from FahimFBA/development"
MERGE (j)-[:IMPLEMENTED_BY]->(c1)
MERGE (j)-[:IMPLEMENTED_BY]->(c2)
MERGE (j)-[:IMPLEMENTED_BY]->(c3)
```

---

## Decisions

**Use AuraDB Free** — zero-ops, no local Docker needed for demo. Sufficient for 72 commits + 28 Jira nodes.

**PRECEDED_BY as fallback** — when Jira linkage absent, commit chain via `PRECEDED_BY` edges still gives traversable graph. GitMind's `traverse_to_root()` uses this.

**Branch as node, not property** — allows querying "all commits on development branch" without full scan.

---

## Rejected Alternatives

| Option | Reason rejected |
|--------|----------------|
| Local Neo4j Docker | Render free tier can't reach localhost:7687 across deploys |
| Embedded graph (networkx) | No Cypher, no graph traversal API, can't serve GitMind queries |
| AuraDB Professional | Cost; free tier sufficient for hackathon scale |

---

## Consequences

- All ingest scripts must use `MERGE` not `CREATE` — idempotent re-runs safe.
- Free tier has no APOC plugin. Avoid `apoc.*` calls in queries.
- Connection drops after ~1hr idle on free tier. Add reconnect logic in driver init.
