> Fictional example for the Granted demo. Every name, number and finding is invented.

# Acme Shop: plan to move the orders database from MySQL 5.7 to PostgreSQL 16

*Written by the migration agent (an AI agent) for the owner of the order service. Sunday 18 October 2026. Draft 1.*

## 1. Summary

The orders database runs on MySQL 5.7. MySQL 5.7 is out of support. The shop pays an extended support fee of $2,900 per month to keep it. [READ]

I read the code, converted the schema and ran checks against a PostgreSQL 16 staging copy. The move is possible, but it is not ready.

- I found 7 problems (F1–F7). One is Critical: nobody has tested a restore or a rollback (F6).
- Five decisions (D1–D5) need you. The most important is the cutover method (D1).
- I recommend CDC replication, a 30-minute window on Sunday 10 January 2027 and a 7-day rollback window.
- The move costs about $27,700 once. It saves about $2,700 per month. Payback takes about 10 months.

One action needs no decision. Fix the binlog upload now (F6).

## 2. Scope and method

### 2.1 The system today

- MySQL 5.7.44 on a managed cloud instance, with one primary and one read replica. [READ]
- 46 tables, 612 GB on disk: 431 GB of data and 181 GB of indexes. [TESTED]
- The largest tables are `order_events` (1.31 billion rows, 288 GB), `order_items` (402 million rows, 97 GB) and `orders` (118 million rows, 64 GB). [TESTED]
- Normal load is 310 writes/s and 2,400 reads/s. The last peak (Black Friday 2025) was 1,450 writes/s. [READ]
- Three clients use the database: the order service (Java 17, Hibernate 5.2.18), the nightly report job (Python) and the support dashboard (read-only). [READ]
- The order service has 38 entity classes and 214 queries. [READ]
- Of the 214 queries, 171 go through the ORM. The other 43 are native SQL. [READ]

### 2.2 Method and evidence labels

1. I read the order service, the report job and the database run book.
2. I loaded a 10% sample into a PostgreSQL 16 staging instance.
3. I ran all 43 native SQL queries and the full service test suite against staging.
4. I restored last night's MySQL backup to a scratch instance.
5. I ran a 30-minute load test at 600 writes/s.

Each claim has a label. **TESTED**: a script or a test ran and its output is saved. **READ**: found in the code or a document. **INFERRED**: a conclusion from reasoning, not tested. **UNVERIFIED**: not checked. **OWNER-ONLY**: only you can do this. Appendix A lists each check.

## 3. Findings

### 3.1 F1 — Three queries use MySQL-only syntax

- [READ] 3 of the 43 native SQL queries use syntax that PostgreSQL does not accept.
- [TESTED] In staging, 40 queries pass. These 3 fail with a syntax error.

| Query | Where | MySQL form | PostgreSQL form |
|---|---|---|---|
| Q1 cart upsert | `CartRepository.upsertLine` | `ON DUPLICATE KEY UPDATE qty = qty + VALUES(qty)` | `ON CONFLICT (cart_id, sku) DO UPDATE SET qty = cart_lines.qty + EXCLUDED.qty` |
| Q2 SKU list | `NightlyReport.skuList` | `GROUP_CONCAT(sku ORDER BY sku SEPARATOR ',')` | `string_agg(sku, ',' ORDER BY sku)` |
| Q3 search page | `OrderSearch.page` | `LIMIT 40, 20` | `LIMIT 20 OFFSET 40` |

[INFERRED] During the rollback window (D3), the same release must run on both databases. Q3 has one form that both accept. Q1 and Q2 do not.

**Proposed action.** Rewrite the 3 queries. Keep a per-database version of Q1 and Q2. The service selects the version at start-up. Effort: 2 days.

**Other option.** Move the 3 queries to the ORM query API. Effort: 4 days. The ORM has no upsert. Q1 becomes a read and then a write, with a race between parallel requests. [INFERRED]

### 3.2 F2 — 11 foreign keys have no index in PostgreSQL

- [READ] MySQL (InnoDB) creates an index for each foreign key. PostgreSQL does not.
- [TESTED] The converted schema has 23 foreign keys. 11 of them have no index.
- [TESTED] In staging, `GET /orders/{id}/items` took 4,100 ms at p95. On MySQL it takes 9 ms. With the index, it took 7 ms.
- [TESTED] In staging, the 11 indexes took 11 minutes to build.
- [INFERRED] At full size, the build takes about 1 h 50 min (10 times the staging time).

**Proposed action.** Build the 11 indexes after the bulk copy and before the CDC catch-up. Effort: 1 day.

**Other option.** Create the indexes before the copy. This is simpler, but the copy becomes about 40% slower (+2 h 10 min). [INFERRED]

### 3.3 F3 — Implicit type casts and zero dates break on PostgreSQL

- [READ] 7 call sites compare a VARCHAR column with a number. An example is `WHERE order_number = ?` with a `long` parameter. MySQL converts the value without a warning.
- [TESTED] PostgreSQL rejects 5 of the 7 with `operator does not exist: character varying = bigint`.
- [UNVERIFIED] The staging tests do not reach the other 2 call sites. I expect the same error.
- [TESTED] 18,204 rows in `orders.shipped_at` and 312 rows in `returns.closed_at` hold the zero date `0000-00-00 00:00:00`. PostgreSQL does not accept this value.
- [READ] The service already reads the zero date as NULL (`zeroDateTimeBehavior=convertToNull`).

**Proposed action.** Bind the 7 parameters as strings. Convert each zero date to NULL during the copy. Effort: 1.5 days. The behaviour of the service does not change.

### 3.4 F4 — DATETIME columns mix UTC and local time

- [READ] A MySQL DATETIME value has no time zone. The schema has 34 DATETIME columns.
- [READ] The service sets its clock to UTC. 31 columns get their values from the service, so they hold UTC.
- [READ] 3 columns get their values from MySQL triggers that call `NOW()`. The server time zone is `SYSTEM`. The host runs on local time (UTC+1, with daylight saving). These 3 columns hold local time: `order_events.created_at`, `stock_reservations.created_at` and `audit_log.changed_at`.
- [TESTED] For the same order, `order_events.created_at` is 1 or 2 hours later than `orders.created_at`.
- [INFERRED] At each autumn clock change, one local hour occurs two times. About 38,000 rows fall in these hours. Their true UTC time is ambiguous.
- [READ] The triggers move to PostgreSQL. There, `now()` returns a `timestamptz`, so new rows are correct.

**Proposed action.** Convert all 34 columns to `timestamptz`. Read the 31 service columns as UTC and the 3 trigger columns as local time. Mark the ambiguous rows in a new column `tz_ambiguous`. Effort: 3 days.

**Other option.** Copy the values as `timestamp without time zone`, with no change. Effort: half a day. The mixed time zones stay, and reports stay wrong by 1–2 hours.

### 3.5 F5 — The ORM must move from version 5.2 to 6.4 first

- [READ] The service uses Hibernate 5.2.18. Its support ended in 2018. Its newest PostgreSQL dialect targets PostgreSQL 9.5.
- [TESTED] With 5.2 against staging, 61 of 1,912 service tests fail. Most failures come from identity columns and the JSON column `orders.metadata`.
- [TESTED] On a trial branch with 6.4, 14 of 1,912 tests fail. Each of the 14 failures traces to F1 or F3.
- [INFERRED] Version 6.4 changes some query results, for example the order of NULL values in sorted lists.

**Proposed action.** Upgrade to 6.4. Release it on MySQL by 17 November 2026, before the peak freeze. Let it run in production for at least one week before the cutover. Then a new error after the cutover has one cause, not two. Effort: 8 days.

### 3.6 F6 — Nobody has tested a restore or a rollback

- [READ] The backup runs each night. It takes a full snapshot at 01:00 and uploads the binlog files for point-in-time recovery.
- [READ] The run book has no record of a restore test.
- [TESTED] I restored last night's snapshot to a scratch instance. The restore took 6 h 12 min. The row counts match on all 46 tables.
- [TESTED] Point-in-time recovery failed. The binlog upload job stopped on 9 October 2026, after a credential change. The bucket has no binlog files after that date.
- [READ] The plan has no rollback path yet. The reverse replication from PostgreSQL to MySQL does not exist.
- [UNVERIFIED] The rollback time of about 20 minutes in section 4.3 is an estimate. Nobody has rehearsed it.

**Why it matters.** Today, a failure loses up to 24 hours of orders. [INFERRED] After the cutover, a failure without a rollback path loses every order since the switch. [INFERRED]

**Proposed action** (effort: 6.5 days).

1. Fix the binlog upload now. Add an alert for the case that no file arrives in 1 hour.
2. Repeat the restore test, with point-in-time recovery.
3. Build the reverse replication.
4. Rehearse a full rollback in staging two times.
5. Make two passed rehearsals a go/no-go gate (D5).

### 3.7 F7 — The move costs about $27,700 once and saves about $2,700 per month

One-time cost [INFERRED]:

| Item | Cost |
|---|---|
| Engineering: 31 engineer-days at $800 | $24,800 |
| Full-size PostgreSQL staging, 6 weeks | $1,900 |
| CDC tool hosting, 6 weeks | $600 |
| Load test tooling | $400 |
| **Total** | **$27,700** |

Engineering days by item: F1 2, F2 1, F3 1.5, F4 3, F5 8, F6 6.5, CDC setup 3, load test 3, rehearsal and cutover 3.

Running cost per month [READ, from the provider price list]:

| Item | Today (MySQL 5.7) | After (PostgreSQL 16) |
|---|---|---|
| Primary and replica | $4,100 | $4,300 |
| Extended support fee | $2,900 | $0 |
| **Total** | **$7,000** | **$4,300** |

- The saving is $2,700 per month. Payback takes about 10 months. [INFERRED]
- The MySQL instances cost about $137 per day while they run in the rollback window. [INFERRED]
- [UNVERIFIED] The price list does not show a contract discount for the shop.

**Proposed action.** Approve a one-time budget of $30,000. This includes a buffer of about 8%.

## 4. The migration plan

### 4.1 Phases

This plan uses the recommended options: D1 b, D2 b, D3 b, D4 a and D5 b.

| Phase | When | Work | Exit condition |
|---|---|---|---|
| 0. Fix now | 19–23 Oct 2026 | Binlog upload fix and restore test (F6) | Point-in-time recovery passes |
| 1. Code | 26 Oct – 13 Nov 2026 | F5, F1 and F3. Release on MySQL by 17 Nov | The release runs one week on MySQL with no new errors |
| 2. Staging | 16 Nov – 18 Dec 2026 | Full-size copy, F2, F4, CDC, reverse CDC, 2 rollback rehearsals, load test | All gates in 4.4 pass |
| Peak freeze | 20 Nov 2026 – 4 Jan 2027 | No production changes | — |
| 3. Replicate | 5–10 Jan 2027 | Bulk copy, index build, CDC catch-up, steady replication | Lag below 5 s for 24 hours. Checksums match |
| 4. Cutover | Sun 10 Jan 2027, 02:00–02:30 | Freeze writes, final checks, switch | Go/no-go checklist passes |
| 5. Rollback window | 10–17 Jan 2027 | Reverse CDC keeps MySQL in sync | No rollback necessary |
| 6. Clean-up | From 18 Jan 2027 | Stop reverse CDC. Keep a final MySQL snapshot for 30 days. Shut down MySQL | — |

### 4.2 Cutover run book (summary)

All times are local time on Sunday 10 January 2027.

1. 01:30 — Go/no-go call (D5). Check the gates in 4.4.
2. 02:00 — Put the shop in maintenance mode. Stop writes to MySQL.
3. 02:02 — Wait until the CDC lag is 0. Expected: less than 1 minute. [INFERRED]
4. 02:05 — Compare row counts and checksums on the 46 tables.
5. 02:10 — Start the reverse replication from PostgreSQL to MySQL.
6. 02:12 — Switch the service connection string to PostgreSQL. Restart the service.
7. 02:15 — Run the smoke tests. Open the shop.
8. 02:15–02:30 — Buffer. If a check fails, start the rollback (4.3).

The write freeze lasts 10 minutes (02:00–02:10). The switch takes 5 minutes (02:10–02:15). Total downtime is about 15 minutes.

### 4.3 Rollback path

During the rollback window, reverse replication copies each change from PostgreSQL to the MySQL standby (`figs/rollback.svg`). If a rollback is necessary:

1. Freeze writes on PostgreSQL.
2. Wait until the reverse replication lag is 0.
3. Switch the connection string back to MySQL. Restart the service.
4. Run the smoke tests. Open the shop.

The expected time is about 20 minutes. [UNVERIFIED] No orders are lost if the lag reaches 0. [INFERRED] The release must run on both databases until the window ends (F1).

### 4.4 Go/no-go gates

- Two full rollback rehearsals passed in staging (F6).
- The point-in-time recovery test passed (F6).
- A load test at 1,450 writes/s for 2 hours passed, with a p95 write latency below 25 ms.
- The CDC lag stayed below 5 s for the last 24 hours.
- Row counts and checksums match on all 46 tables.
- The support team and the on-call database lead are on call.

Load test so far:

- [TESTED] 30 minutes at 600 writes/s. The p95 write latency was 14 ms on PostgreSQL and 11 ms on MySQL. The CDC lag stayed between 1 and 3 s.
- [UNVERIFIED] Nobody has tested the peak rate of 1,450 writes/s.

## 5. Decisions only the owner can make

Each decision is [OWNER-ONLY]. The review form has one card for each decision, with the full options.

| Decision | Options | I recommend | Decide by |
|---|---|---|---|
| D1 Cutover method | a. Big-bang copy (about 8 h 20 min of downtime). b. CDC replication, then a short switch (about 15 min). c. Dual writes from the service (no planned downtime, 15–20 more days of work). | b | 30 Oct 2026 |
| D2 Date and window | a. Sun 15 Nov 2026, 02:00–02:30. b. Sun 10 Jan 2027, 02:00–02:30. c. Tue 12 Jan 2027, 03:00–03:30. | b | 6 Nov 2026 |
| D3 Rollback window | a. 48 hours (about $274). b. 7 days (about $960). c. 30 days (about $7,000). | b | 6 Nov 2026 |
| D4 Schema freeze | a. Full freeze, 5–17 Jan (12 days). b. Short freeze, from 1 day before to 2 days after the cutover. c. No freeze, with a dual-apply procedure. | a | 11 Dec 2026 |
| D5 Approvals | a. You approve the go and each rollback. b. You approve the go. The on-call database lead can start a rollback alone. c. A change board approves on the Thursday before. | b | 18 Dec 2026 |

Facts behind the options:

- [TESTED] The bulk copy of the 10% sample took 33 minutes. [INFERRED] At full size, D1 option a needs 5 h 30 min to copy, 1 h 50 min for indexes, 40 min for checks and 20 min to switch.
- [READ] On a typical Sunday, about 22,000 orders arrive between 02:00 and 10:20.
- [READ] Sunday 02:00–02:30 has the lowest traffic: 41 orders/min. Tuesday 03:00–03:30 has 57 orders/min.
- [INFERRED] The work for the move is 31 engineer-days. With 2 engineers, that is about 16 working days.
- [INFERRED] D2 option a leaves no slack. Its rollback window ends on 22 November, after the peak freeze starts. D2 option b delays about $5,400 of savings.
- [READ] The settlement job runs each Friday night. The invoice run is on the 1st of each month.
- [TESTED] In staging, the CDC tool stopped when I added a column on MySQL only.
- [READ] The team merged 9 schema changes in the last 90 days, about 0.7 per week.

## 6. What this review did not cover

- The analytics replica. 12 ETL jobs read the MySQL replica directly. I did not read them. [UNVERIFIED]
- The peak load of 1,450 writes/s (see 4.4). [UNVERIFIED]
- Security: database roles, secrets and network rules for the new instances.
- The payment database. It is a separate system.
- Data residency and the legal review of the new instance region.

## 7. Next steps

1. Fix the binlog upload (F6). Do it now. It needs no decision.
2. Review F1–F7 and make the decisions D1–D5 in the review form.
3. I will update the plan from your answers and send round 2.

## Appendix A. Check log

| Check | What | Label | Result |
|---|---|---|---|
| C01 | Count tables and sizes | TESTED | 46 tables, 612 GB |
| C02 | Search native SQL for MySQL-only syntax | READ | 3 hits (F1) |
| C03 | Run the 43 native queries on staging | TESTED | 40 pass, 3 fail (F1) |
| C04 | Compare foreign keys with indexes | TESTED | 11 of 23 have no index (F2) |
| C05 | p95 of the items endpoint | TESTED | 4,100 ms without the index, 7 ms with it (F2) |
| C06 | Find VARCHAR-to-number comparisons | READ | 7 call sites (F3) |
| C07 | Count zero dates | TESTED | 18,516 rows (F3) |
| C08 | Trace the writers of DATETIME columns | READ | 31 UTC, 3 local time (F4) |
| C09 | Service tests on staging, ORM 5.2 and 6.4 | TESTED | 61 and 14 of 1,912 fail (F5) |
| C10 | Snapshot restore | TESTED | 6 h 12 min, row counts match (F6) |
| C11 | Point-in-time recovery | TESTED | Failed. No binlog files after 9 Oct (F6) |
| C12 | Load test, 600 writes/s, 30 min | TESTED | p95 14 ms, CDC lag 1–3 s (4.4) |
| C13 | Bulk copy and index build, 10% sample | TESTED | 33 min and 11 min (F2, D1) |
| C14 | Add a column on MySQL only during CDC | TESTED | CDC stops with a mapping error (D4) |
| C15 | ENUM columns, stored procedures, scheduled events | READ | 4 ENUM columns map to enum types. No procedures or events. No problem |
