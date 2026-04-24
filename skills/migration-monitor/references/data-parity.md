# Data Parity Reference

## Purpose

This reference drives Phase 6 database data parity checks. It applies to SQL databases migrated from VMware-hosted hosts to Amazon RDS or Aurora. Falcon runs read-only queries against each source/target pair, compares table inventory, row counts, and column schemas, and emits structured divergences. Out of scope: NoSQL stores (MongoDB, DynamoDB), file-level hash comparisons, and data-content checksums. Phase 6 does not modify any database; only `SELECT` queries are issued.

---

## Supported engines

Phase 6 supports the following SQL engines:

| Engine | Minimum version | CLI tool |
|--------|----------------|----------|
| MySQL | 5.7+ | `mysql` |
| PostgreSQL | 12+ | `psql` |
| SQL Server | 2017+ | `sqlcmd` |
| Oracle | — | DEFERRED to v1.1 (requires proprietary OCI client; not bundled) |

Falcon invokes the CLI tools listed above directly. They must be present on Falcon's host. If a required CLI is absent, Falcon emits `data.engine-unavailable` (blocked) for that pair and skips all further checks for that pair.

NoSQL stores (MongoDB, DynamoDB, Redis, Cassandra, etc.) are out of scope for Phase 6. No attempt is made to query, compare, or report on them.

---

## URI format and credential handling

### URI formats

Database connection URIs follow these forms:

- MySQL: `mysql://user@host:3306/db`
- PostgreSQL: `postgresql://user@host:5432/db`
- SQL Server: `mssql://user@host:1433/db`

### Credential handling

Passwords are sourced exclusively from environment variables:

- Source DB password: `SOURCE_DB_PASSWORD`
- Target DB password: `TARGET_DB_PASSWORD`

Passwords MUST NOT be constructed as shell positional arguments visible in `ps` output whenever a safe alternative exists. The patterns below show the preferred form per engine.

### Concrete invocation patterns

#### MySQL

```bash
# MySQL — MYSQL_PWD env var is the required form (not visible in ps -ef)
MYSQL_PWD="$DB_PASSWORD" \
mysql -h "$HOST" -P "$PORT" -u "$USER" \
  -D "$DB" \
  -N -B \
  --connect-timeout=10 \
  -e "SELECT ...;"
```

`MYSQL_PWD` is the REQUIRED form for passing the MySQL password. Unlike `--password=`, `MYSQL_PWD` is not visible in `ps -ef` output (it is only readable via `/proc/<pid>/environ` by the same user or root). Using `--password=` on the command line IS visible in `ps -ef` and is strongly discouraged. The `--password=` flag is a fallback only for environments that cannot set environment variables at all; if it must be used, the exposure must be explicitly accepted by the user.

#### PostgreSQL

```bash
# PostgreSQL — PGPASSWORD + PGCONNECT_TIMEOUT env vars
PGPASSWORD="$DB_PASSWORD" \
PGCONNECT_TIMEOUT=10 \
psql -h "$HOST" -p "$PORT" -U "$USER" \
     -d "$DB" \
     -t -A \
     -w \
     -c "SELECT ...;"
```

`-t` suppresses column headers and row counts; `-A` removes alignment padding; `-w` disables password prompt (fails fast if `PGPASSWORD` is not set).

#### SQL Server

```bash
# SQL Server — SQLCMDPASSWORD env var is the required form
SQLCMDPASSWORD="$DB_PASSWORD" \
sqlcmd -S "$HOST,$PORT" -U "$USER" \
       -d "$DB" \
       -h -1 -W \
       -l 10 \
       -Q "SELECT ...;"
```

`SQLCMDPASSWORD` is the REQUIRED form. It has been supported since SQL Server 2005; all in-scope versions (2017+) support it. Setting `SQLCMDPASSWORD` avoids exposing the password in `ps` output. The `-P` flag is a discouraged fallback only for legacy environments where environment variables cannot be set; its use must be explicitly accepted by the user.

### Connection timeout summary

| Engine | Timeout flag | Value |
|--------|-------------|-------|
| MySQL | `--connect-timeout=10` | 10 seconds |
| PostgreSQL | implicit via `-w` (no prompt); set `PGCONNECT_TIMEOUT=10` | 10 seconds |
| SQL Server | `-l 10` | 10 seconds |

---

## Table inventory queries

Falcon queries the information schema of each engine to enumerate base tables. The result is sorted and compared between source and target.

### MySQL

```sql
SELECT table_name
  FROM information_schema.tables
 WHERE table_schema = DATABASE()
   AND table_type = 'BASE TABLE'
 ORDER BY table_name;
```

`DATABASE()` returns the currently selected schema, scoping the query to the connected database only.

### PostgreSQL

```sql
SELECT schemaname || '.' || tablename AS qualified_name
  FROM pg_catalog.pg_tables
 WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
 ORDER BY schemaname, tablename;
```

System schemas are excluded. The result is a fully qualified `schema.table` name for unambiguous comparison across databases that may share table names in different schemas.

### SQL Server

```sql
SELECT TABLE_SCHEMA + '.' + TABLE_NAME AS qualified_name
  FROM INFORMATION_SCHEMA.TABLES
 WHERE TABLE_TYPE = 'BASE TABLE'
   AND TABLE_SCHEMA NOT IN ('sys', 'INFORMATION_SCHEMA')
 ORDER BY TABLE_SCHEMA, TABLE_NAME;
```

System schemas `sys` and `INFORMATION_SCHEMA` are excluded. The result is fully qualified `schema.table`.

### Cross-engine qualifier normalization

When source and target engines differ (e.g., MySQL source migrated to PostgreSQL target), Falcon normalizes table names before comparison:

- If the source is MySQL (no schema), add a default schema prefix to match the target convention (PostgreSQL typically uses `public`, SQL Server uses `dbo`). Falcon uses `--target-default-schema <name>` (default: `public` for PostgreSQL, `dbo` for SQL Server) to pick the prefix.
- For same-engine pairs (e.g., PostgreSQL source → PostgreSQL target in RDS), schema identity is preserved — `public.users` on source must match `public.users` on target, not `custom_schema.users`.
- Cross-engine comparisons are lossy in schema identity; Falcon emits an informational note at the top of the data-parity section of the report explaining the normalization applied.

### Comparison logic

Falcon parses the output of each query into a sorted list of table names. It then computes set differences:

- **Tables only on source (missing on target):** emit `data.table.missing-on-target` — **critical**. The migration is incomplete; data that exists on the source has no landing table on the target.
- **Tables only on target (extra on target):** emit `data.table.extra-on-target` — **minor**. Common during incremental schema evolution; the table was created after the migration baseline or by the migration tooling itself.

Example divergences:

```json
{
  "id": "data-001",
  "phase": "data",
  "type": "data.table.missing-on-target",
  "severity": "critical",
  "description": "Table 'orders' exists on source DB but has no counterpart on target RDS",
  "evidence": {"table": "orders", "source_engine": "mysql", "target_engine": "mysql"}
}
```

```json
{
  "id": "data-002",
  "phase": "data",
  "type": "data.table.extra-on-target",
  "severity": "minor",
  "description": "Table 'schema_migrations' exists on target RDS but not on source DB",
  "evidence": {"table": "schema_migrations", "source_engine": "mysql", "target_engine": "mysql"}
}
```

---

## Row count queries

### Base form (all engines)

```sql
SELECT COUNT(*) FROM <qualified_table_name>;
```

`<qualified_table_name>` is the table name as returned by the inventory query, already qualified with schema where applicable.

### With cutoff (for CDC-in-flight migrations)

When the user provides `--cutoff-field` and `--cutoff-value`, Falcon issues the bounded form to exclude rows written after the cutoff point. This accounts for rows that were written to the source after the replication snapshot was taken but before CDC has fully propagated them to the target.

```sql
SELECT COUNT(*) FROM <qualified_table_name>
 WHERE <cutoff_field> < <cutoff_value>;
```

#### Cutoff value literal format per engine

| Engine | Cutoff literal format | Example |
|--------|----------------------|---------|
| MySQL | Quoted string `'YYYY-MM-DD HH:MM:SS'` | `'2026-04-24 00:00:00'` |
| PostgreSQL | Timestamptz cast `'2026-04-24T00:00:00Z'::timestamptz` or plain string `'2026-04-24 00:00:00'` (adjust per column type) | `'2026-04-24T00:00:00Z'::timestamptz` |
| SQL Server | Quoted string `'2026-04-24 00:00:00'` (adjust per column type) | `'2026-04-24 00:00:00'` |

### Cutoff value source

The cutoff value comes from the USER at Phase 6 invocation time, not from automatic derivation. Users who know their migration's CDC replication is still running can specify a timestamp before the cutover started (e.g., `--cutoff-field updated_at --cutoff-value '2026-04-20 00:00:00'`) to restrict counts to rows guaranteed to exist on both sides.

If no cutoff is provided, Falcon uses the raw `COUNT(*)` queries. Without a cutoff, replication lag can cause spurious `data.rowcount.deficit` divergences — Falcon flags this risk in the report if the user has reported an active CDC state but no cutoff field.

Falcon MUST NOT auto-derive a cutoff (e.g., "now minus 1 hour") because (a) clock skew between source and target is unknowable and (b) the correct cutoff depends on application-level write patterns that Falcon cannot inspect.

### Tolerance

- **Default tolerance:** within 1%. Divergence fires when `abs(source_count - target_count) / source_count > 0.01`. A 1% band accounts for rows in-flight during CDC replication and normal write activity during the check window.
- **Strict mode** (`--strict-row-count` flag): 0% tolerance. Any count difference triggers a divergence, regardless of magnitude.

#### Empty-table handling

The formula above divides by `source_count` and crashes when `source_count == 0`. Falcon applies the following rules before evaluating the tolerance formula:

- `source_count == 0 AND target_count == 0`: no divergence (tables are equivalent).
- `source_count == 0 AND target_count > 0`: emit `data.rowcount.surplus` with `deficit_pct: null`, severity minor.
- `source_count > 0 AND target_count == 0`: emit `data.rowcount.deficit` with `deficit_pct: 1.0` (100% deficit), severity critical.
- `source_count > 0 AND target_count > 0`: compute `delta_pct = abs(source_count - target_count) / source_count`. Apply normal tolerance rules (1% default / 10% critical).

### Divergence classification

| Condition | Type | Severity |
|-----------|------|----------|
| Target has fewer rows; deficit 1–10% | `data.rowcount.deficit` | major |
| Target has fewer rows; deficit > 10% | `data.rowcount.deficit` | critical (significant data loss risk) |
| Target has more rows; surplus > 1% | `data.rowcount.surplus` | minor |

Example divergences:

```json
{
  "id": "data-010",
  "phase": "data",
  "type": "data.rowcount.deficit",
  "severity": "critical",
  "description": "Table 'events' has 1,000,000 rows on source but 850,000 on target (deficit 15%, exceeds 10% threshold)",
  "evidence": {
    "table": "events",
    "source_count": 1000000,
    "target_count": 850000,
    "deficit_pct": 15.0,
    "cutoff_field": null,
    "cutoff_value": null
  }
}
```

```json
{
  "id": "data-011",
  "phase": "data",
  "type": "data.rowcount.surplus",
  "severity": "minor",
  "description": "Table 'audit_log' has 500,000 rows on source but 510,000 on target (surplus 2%); may indicate duplicate replication or replication restart",
  "evidence": {
    "table": "audit_log",
    "source_count": 500000,
    "target_count": 510000,
    "surplus_pct": 2.0,
    "cutoff_field": null,
    "cutoff_value": null
  }
}
```

---

## Column schema queries

Falcon queries column metadata for each table that exists on both source and target. Column schema checks are skipped for tables that have already triggered `data.table.missing-on-target` (no target table to query).

### MySQL

```sql
SELECT column_name, ordinal_position, data_type
  FROM information_schema.columns
 WHERE table_schema = DATABASE()
   AND table_name = '<table>'
 ORDER BY ordinal_position;
```

### PostgreSQL

```sql
SELECT column_name, ordinal_position, data_type
  FROM information_schema.columns
 WHERE table_schema = '<schema>'
   AND table_name = '<table>'
 ORDER BY ordinal_position;
```

`<schema>` is the schema component of the qualified table name from the inventory query.

### SQL Server

```sql
SELECT COLUMN_NAME, ORDINAL_POSITION, DATA_TYPE
  FROM INFORMATION_SCHEMA.COLUMNS
 WHERE TABLE_SCHEMA = '<schema>'
   AND TABLE_NAME = '<table>'
 ORDER BY ORDINAL_POSITION;
```

### Comparison rules

Falcon compares column metadata as follows:

1. **Column names:** compared as a sorted set. A column present on the source but absent from the target = `data.column.missing-on-target` (critical). A column present on the target but absent from the source = `data.column.extra-on-target` (minor).
2. **Ordinal positions:** for columns that exist on both sides, compare `ordinal_position`. A mismatch (same column name, different position) = `data.column.position-mismatch` (minor). Column order affects some ORM query patterns and `SELECT *` result ordering.
3. **Column count:** an implicit check derived from the name comparison. Differences surface as missing or extra column divergences.
4. **Data types:** data type names (`data_type` / `DATA_TYPE`) are recorded in evidence for audit purposes but are NOT used for divergence classification. Platform drift is expected: MySQL `INT` maps to PostgreSQL `integer`, SQL Server `nvarchar` maps to PostgreSQL `character varying`, etc. Comparing data types across heterogeneous engines produces false positives.

Example divergences:

```json
{
  "id": "data-020",
  "phase": "data",
  "type": "data.column.missing-on-target",
  "severity": "critical",
  "description": "Column 'deleted_at' exists in source table 'users' but is absent from the target table",
  "evidence": {
    "table": "users",
    "column": "deleted_at",
    "source_ordinal": 12,
    "source_data_type": "datetime"
  }
}
```

```json
{
  "id": "data-021",
  "phase": "data",
  "type": "data.column.extra-on-target",
  "severity": "minor",
  "description": "Column 'created_by_arn' exists in target table 'users' but is absent from the source table",
  "evidence": {
    "table": "users",
    "column": "created_by_arn",
    "target_ordinal": 15,
    "target_data_type": "character varying"
  }
}
```

```json
{
  "id": "data-022",
  "phase": "data",
  "type": "data.column.position-mismatch",
  "severity": "minor",
  "description": "Column 'email' in table 'users' is at ordinal position 3 on source but position 5 on target",
  "evidence": {
    "table": "users",
    "column": "email",
    "source_ordinal": 3,
    "target_ordinal": 5,
    "source_data_type": "varchar",
    "target_data_type": "character varying"
  }
}
```

---

## Divergence classification matrix

Summary of all divergence types produced by Phase 6, their default severity, and escalation conditions.

| Type | Severity | Escalation |
|------|----------|-----------|
| `data.table.missing-on-target` | critical | — |
| `data.table.extra-on-target` | minor | — |
| `data.rowcount.deficit` | major (deficit 1–10%) | critical (deficit > 10%) |
| `data.rowcount.surplus` | minor | — |
| `data.column.missing-on-target` | critical | — |
| `data.column.extra-on-target` | minor | — |
| `data.column.position-mismatch` | minor | — |
| `data.engine-unavailable` | blocked | — (CLI tool missing on Falcon's host; entire pair is skipped) |
| `data.connection-failure` | critical | — (cannot reach source or target DB; pair is skipped) |
| `data.query-timeout` | major | critical (if the timeout recurs across multiple tables in the same pair) |

### `data.engine-unavailable`

Emitted when the CLI tool for a required engine (`mysql`, `psql`, or `sqlcmd`) is not found on Falcon's host. All three check types (inventory, row count, column schema) are skipped for the affected pair. Example:

```json
{
  "id": "data-030",
  "phase": "data",
  "type": "data.engine-unavailable",
  "severity": "blocked",
  "description": "CLI tool 'sqlcmd' not found on Falcon host; SQL Server pair skipped",
  "evidence": {"engine": "mssql", "cli_tool": "sqlcmd", "pair_id": "pair-db-sqlserver-01"}
}
```

### `data.connection-failure`

Emitted when the CLI tool is present but a connection to the source or target DB cannot be established (auth failure, network unreachable, hostname resolution failure, etc.). Example:

```json
{
  "id": "data-031",
  "phase": "data",
  "type": "data.connection-failure",
  "severity": "critical",
  "description": "Cannot connect to target RDS endpoint 'prod-db.xxx.us-east-1.rds.amazonaws.com:5432'",
  "evidence": {
    "side": "target",
    "host": "prod-db.xxx.us-east-1.rds.amazonaws.com",
    "port": 5432,
    "engine": "postgresql",
    "error_sample": "could not connect to server: Connection refused"
  }
}
```

### `data.query-timeout`

Emitted when a query exceeds 5 minutes (see Exclusion and safety). Example:

```json
{
  "id": "data-032",
  "phase": "data",
  "type": "data.query-timeout",
  "severity": "major",
  "description": "Row count query on table 'events' exceeded 5-minute timeout on source DB",
  "evidence": {
    "table": "events",
    "side": "source",
    "timeout_seconds": 300,
    "engine": "mysql"
  }
}
```

---

## Exclusion and safety

### Table exclusion

Use `--exclude-table <regex>` to skip tables whose names match the provided regular expression. This is intended for volatile tables that churn too rapidly for a consistent count to be meaningful:

- Session tables (e.g., `user_sessions`, `php_sessions`)
- Audit logs that receive continuous writes during the check window
- Temporary or staging tables populated and truncated by batch jobs

Multiple `--exclude-table` flags may be specified. Each pattern is applied to the fully qualified table name (e.g., `public.user_sessions`). Excluded tables are recorded in the Phase 6 report as skipped rather than divergent.

### Read-only enforcement

Phase 6 issues only `SELECT` queries. Falcon MUST validate every SQL string before emission to the CLI tool. The following verbs are disallowed — Falcon rejects any command string containing them (case-insensitive, whole-word match):

- **Data modification:** `INSERT`, `UPDATE`, `DELETE`, `MERGE`
- **Schema modification:** `DROP`, `CREATE`, `ALTER`, `TRUNCATE`
- **Privilege modification:** `GRANT`, `REVOKE`
- **Execution:** `CALL`, `DO`, `EXECUTE`
- **Output redirection:** `SELECT ... INTO` (the `INTO` keyword immediately following a `SELECT` column list)

Allowed forms:

- `SELECT` (simple column or expression select)
- `SELECT COUNT(*)`
- `SELECT ... FROM information_schema.*` (metadata queries)
- `SELECT ... FROM pg_catalog.*` (PostgreSQL system catalog queries)
- `SELECT ... FROM INFORMATION_SCHEMA.*` (SQL Server uppercase form)

If a disallowed verb is detected, Falcon MUST abort that query, log the rejected command, and emit an internal error to the Phase 6 log. It does NOT emit a divergence for the rejected command itself.

### Connection timeouts

All CLI invocations include connection timeout settings to prevent hung connections from blocking Phase 6 indefinitely:

- MySQL: `--connect-timeout=10` (10-second connect timeout)
- PostgreSQL: set `PGCONNECT_TIMEOUT=10` in the environment before invoking `psql`
- SQL Server: `-l 10` (10-second login timeout)

### Query execution timeout

If any single query runs for more than 5 minutes (300 seconds), Falcon MUST abort the query and emit `data.query-timeout` as a major divergence. If the timeout recurs across multiple tables in the same DB pair, escalate to critical.

Implementation: wrap the CLI invocation with `timeout 300 <cli-command>`. If `timeout` exits with code 124, treat as a timeout event.

### Large-table sampling warning

For tables with more than 10 million rows, a full `COUNT(*)` can be slow and may cause brief metadata locks on some engines (particularly MySQL with certain storage engines under high concurrency). Falcon does NOT sample large tables — accurate counts are required for meaningful parity assessment. However, Falcon MUST emit a warning note in the Phase 6 report indicating which tables exceeded 10M rows and that count accuracy was prioritized over query speed. This note is informational only and does not affect divergence classification.
