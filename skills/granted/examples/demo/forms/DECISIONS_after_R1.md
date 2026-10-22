# Decisions after round 1

From the answers to R1.0 (`R1/answers/answers_R1.0_2026-10-22T1526Z.json`). Written 2026-10-22T15:26:45Z.

## Confirmed (9)

| Item | Decision | What will happen | Since | Comment |
|---|---|---|---|---|
| **F1** Three native queries use MySQL-only syntax and fail on PostgreSQL | Option a: Rewrite in SQL, with a per-database version of Q1 and Q2 | Rewrite in SQL, with a per-database version of Q1 and Q2: Effort: 2 days. Risk: low. The 3 queries keep their speed. The two versions of Q1 and Q2 stay in the code until the rollback window ends. Then remove the MySQL versions. | R1.0 |  |
| **F2** 11 foreign keys have no index in PostgreSQL, so one endpoint is 450 times slower | Option a: Build the indexes after the copy, before the catch-up | Build the indexes after the copy, before the catch-up: Effort: 1 day. The copy keeps its speed (about 5 h 30 min). The build adds about 1 h 50 min. With CDC (D1 option b), the shop stays open during both steps. | R1.0 |  |
| **F3** Implicit type casts and zero dates break on PostgreSQL | Agree: do the proposed action | Bind the 7 parameters as strings. Add a test for each of the 7 call sites. Set the copy tool to write NULL for each zero date. Effort: 1.5 days. The service already reads the zero date as NULL, so its behaviour does not change. | R1.0 |  |
| **F5** The ORM must move from version 5.2 to 6.4 and run on MySQL before the cutover | Agree: do the proposed action | Upgrade to 6.4 on a branch. Fix the 14 failing tests with F1 and F3. Release on MySQL by 17 November 2026, before the peak freeze starts on 20 November. Watch the error rate for one week. Effort: 8 days. | R1.0 |  |
| **F6** Nobody has tested a restore or a rollback, and point-in-time recovery fails today | Agree: do the proposed action | Fix the binlog upload now. Add an alert for the case that no file arrives in 1 hour. Repeat the restore test, with point-in-time recovery. Build the reverse replication. Rehearse a full rollback in staging two times. Make two passed rehearsals a go/no-go gate (D5). Effort: 6.5 days. | R1.0 |  |
| **D1** Select the cutover method for the orders database | Option b: CDC replication, then a 15-minute switch | CDC replication, then a 15-minute switch: Downtime: about 15 minutes (10 min write freeze, 5 min switch). Effort: 3 days for CDC setup, in the estimate (F7). Risk: low to medium. A CDC mapping error can copy wrong values. Row counts and checksums find them before the switch. | R1.0 |  |
| **D4** Select the freeze period for schema changes around the cutover | Option a: Full freeze from the start of the copy to the end of the rollback window | Full freeze from the start of the copy to the end of the rollback window: Dates: 5–17 January 2027 (12 days), with D3 option b. Cost: about 1–2 schema changes wait. Risk: low. | R1.0 |  |
| **D5** Name who approves the go and who can start a rollback | Option b: You approve the go. The on-call database lead can start a rollback alone | You approve the go. The on-call database lead can start a rollback alone: Effect: you approve the go at 01:30 against the checklist. After 02:00, the database lead starts a rollback without a call. Risk: low. | R1.0 | "Good. Also invite the support lead to the go call as an observer." |
| **N1** This review did not cover analytics jobs, peak load, security or the payment database | Noted | Five areas are outside this review. Confirm that you read the list, or name an owner for a gap in your comment. | R1.0 |  |

## Needs a decision (4)

| Item | Why it is open | Your answer | Your comment |
|---|---|---|---|
| **F4** Three DATETIME columns hold local time, and the other 31 hold UTC | You answered: Agree with the problem, but change the action | Agree with the problem, but change the action | "Convert to UTC, but keep the old columns for 30 days, so the reports team can compare." |
| **F7** The move costs about $27,700 once and saves about $2,700 per month | You answered: I need more information | I need more information | "What are the $2,900 of costs outside engineering? Are they one-time costs?" |
| **D2** Select the date and the 30-minute downtime window for the cutover | You selected Other | Option other: Other (see the comment) | "Use Sunday 17 January 2027 at 02:00. The winter sale ends on 15 January." |
| **D3** Select how long to keep the MySQL standby for a rollback | You were unsure | Option b: 7 days |  |
