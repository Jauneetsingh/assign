**Part 1: Presenting Your Take-Home Assignment (20–25 min)**
They want you to lead this section, presenting as if they are the customer (ACME). Structure your walkthrough like this:
Suggested Presentation Flow
Opening (1–2 min): "Hi ACME team, thanks for reaching out. I've reviewed all three issues in your ticket and I'd like to walk you through my findings and recommendations one by one."
**Issue 1 — Database Limit (3–4 min)**

Explain the difference between hard limit (max_database_num_to_throw in server config) and soft limit (RBAC approach — pre-create databases, revoke CREATE DATABASE).
Walk through why a server restart is needed for the hard limit.
Show the validation query against system.server_settings.

**Issue 2 — RBAC Users (5–7 min)**

Walk through admin user creation and GRANT ALL approach with WITH GRANT OPTION.
Explain why you matched the default user's permissions (they asked for "same permission level as default").
Developer profile: explain CREATE SETTINGS PROFILE for max_execution_time and max_memory_usage.
Row policy on system.tables — explain how CREATE ROW POLICY filters rows transparently.
Mention the SELECT sleep(1) validation technique.

**Issue 3 — Keeper Configuration (5–7 min)**

This is the most critical section — walk through your debugging process.
Show the original config, point out the <listen_host> was inside <keeper_server>.
Explain the root cause: Keeper ignores listen_host at that nesting level and defaults to 127.0.0.1.
Show the log line that confirms the issue.
Present the fix: move <listen_host> to root <clickhouse> level.
Show verification via system.zookeeper_connection.
Add the broader context: what Keeper does, Raft consensus, production best practices.

Closing (1–2 min): "Before I wrap up, I wanted to flag a couple of follow-up items for your team..." — shows proactiveness.
Potential Follow-Up Questions They May Ask About Your Assignment

**"Why did you choose a hard limit over a soft limit for the database restriction?"**

Good answer: Explain you showed the hard limit because ACME specifically asked for it, but mention the soft limit (RBAC) is more flexible and doesn't require a restart. In production, a combination of both is ideal.

**
**"What happens if someone tries to create a 4th database with the hard limit in place?"****

ClickHouse throws an error. The setting max_database_num_to_throw causes the server to reject the CREATE DATABASE statement once the count reaches 3.


"Why did you grant each privilege individually to the admin user instead of GRANT ALL?"

In ClickHouse, GRANT ALL does exist but there are nuances. Some privileges like GRANT OPTION and access management must be granted explicitly. You matched the default user's grants using SHOW GRANTS to ensure parity.


"What if the developer user needs to see some system tables but not all?"

You can create more granular row policies or grant SELECT on specific system tables. Row policies can use complex conditions beyond just database name.


"Could you have used quotas instead of settings profiles for the developer?"

Quotas and settings profiles serve different purposes. Quotas limit aggregate usage over time periods (e.g., total queries per hour). Settings profiles limit per-query resources. For "500ms max per query," a settings profile is the correct tool.


**"What if the Keeper node goes down in production?"**

Single-node Keeper means total loss of coordination. ReplicatedMergeTree tables will go read-only. This is why you recommended 3+ nodes in production with Raft quorum.


**"How did you reproduce the Keeper issue?"**

Walk through your Docker setup: created a Docker network, launched Keeper with the original (broken) config, checked logs, saw the 127.0.0.1 binding, then fixed it and verified.




**Part 2: Technical Questions & Scenarios (20 min)**
These are the types of questions a ClickHouse Support team would ask. Study all of them.
Category A: ClickHouse Architecture & Internals
Q: What is a MergeTree engine and why is it the default choice?
MergeTree is the foundational table engine in ClickHouse. It stores data in sorted parts on disk, and background merges consolidate small parts into larger ones. It supports primary key indexing (sparse index), partitioning, and TTL. Variants include ReplicatedMergeTree (adds replication via Keeper), SummingMergeTree (auto-sums on merge), AggregatingMergeTree (stores aggregation states), ReplacingMergeTree (deduplicates on merge), and CollapsingMergeTree (tracks row state changes).
Q: How does ClickHouse's sparse primary index differ from a B-tree index?
ClickHouse doesn't index every row. It stores one index entry per granule (default 8192 rows). This means the index is small enough to fit entirely in memory. Lookups find the granule range, then scan within granules. It's optimized for analytical range queries, not point lookups.
Q: What is a granule?
A granule is the smallest indivisible unit of data that ClickHouse reads. Default size is 8192 rows. When ClickHouse processes a query, it skips entire granules that don't match the WHERE clause using the sparse index.
Q: Explain partitioning in ClickHouse.
Partitioning splits data into logical parts (e.g., by month). Each partition is stored independently. Benefits: partition-level operations (DROP PARTITION, DETACH/ATTACH), efficient TTL, better data management. Caution: too many partitions (>1000) creates overhead — keep partitions coarse.
Q: What is the difference between ReplicatedMergeTree and MergeTree?
ReplicatedMergeTree adds multi-node replication via ClickHouse Keeper (or ZooKeeper). It stores replication metadata in Keeper. All replicas are equal (multi-master for reads). Inserts go to any replica and propagate. MergeTree is single-node only.
Q: What are materialized views in ClickHouse?
A materialized view is a trigger that runs a query on INSERT to the source table and writes results to a target table. It's not a cached query — it's an incremental transformation pipeline. Useful for pre-aggregation, denormalization, and routing data to multiple destinations.
Q: What happens during a merge operation?
Background merges combine smaller data parts into larger ones within a partition. During merge, ClickHouse reads multiple parts, applies the MergeTree variant logic (summing, replacing, etc.), writes a new larger part, and marks old parts for deletion. Merges are essential — without them, query performance degrades as part count grows.
Category B: ClickHouse Operations & Troubleshooting
Q: A customer reports slow queries. How do you troubleshoot?

**Check system.query_log for the specific query — look at read_rows, read_bytes, memory_usage, query_duration_ms.**
Run EXPLAIN or EXPLAIN PIPELINE to see the query plan.
Check if primary key is being used (look for PrimaryKeyCondition in the plan).
Check partition pruning — is the WHERE clause hitting the partition key?
Look at system.parts — too many small parts means merges are behind.
Check system.merges for active merge operations consuming resources.
Check system.processes for concurrent queries competing for resources.
Check memory and CPU via system.metrics and system.asynchronous_metrics.

**Q: A customer says "my inserts are slow." What do you look at?**

Are they doing tiny single-row inserts? ClickHouse prefers batches of 10,000+ rows.
Check system.parts — too many parts means the MergeTree is overwhelmed (the "too many parts" error).
Check if max_parts_in_total or parts_to_throw_insert is being hit.
Is there a materialized view slowing down inserts?
Network issues if inserting from a remote client.
Check system.mutations — heavy mutations can compete with inserts.

**Q: How would you handle a "Too many parts" error?**
This means the number of active parts exceeds the threshold (parts_to_throw_insert, default 300 per partition). Solutions: batch inserts into larger chunks, reduce partition granularity, check if merges are keeping up (look at system.merges), increase max_threads for merges, use OPTIMIZE TABLE as a temporary fix, check disk I/O.
Q: Customer gets "Memory limit exceeded" — how to troubleshoot?

**Check max_memory_usage setting for the user/profile.**
Look at system.query_log for peak_memory_usage of the failing query.
Consider: is the query doing a large JOIN or GROUP BY? ClickHouse holds intermediate data in memory.
Solutions: increase memory limit, use max_bytes_before_external_group_by or max_bytes_before_external_sort to spill to disk, optimize the query to process less data.

**Q: How do you approach a customer who says "ClickHouse is not working"?**
This tests your support process skills:

**Clarify the issue — "not working" could mean anything. Ask: what specifically fails, error messages, when did it start, what changed?**
Check logs: clickhouse-server.log and clickhouse-server.err.log.
Check server status: SELECT 1, system tables, system.crashes.
Check disk space, memory, CPU.
Reproduce the issue in a controlled environment if possible.

**Q: A customer wants to migrate from PostgreSQL to ClickHouse. What advice do you give?**

ClickHouse is columnar OLAP, not a replacement for OLTP. Don't use it for transactional workloads.
Denormalize data — ClickHouse doesn't do JOINs as efficiently as row-stores.
Use bulk inserts, not single-row INSERT.
Think about your query patterns first, then design your schema (sort order, partition key).
Use the PostgreSQL table engine or postgresql() function for initial data migration.

**Category C: ClickHouse Configuration & Security**
Q: What is the difference between users.xml and SQL-driven access control?
users.xml is file-based (static, needs restart). SQL-driven RBAC uses CREATE USER, GRANT, etc. (dynamic, no restart). Best practice is SQL-driven RBAC for production. The access_management setting must be enabled for the initial user to use SQL RBAC.
Q: Explain the ClickHouse configuration hierarchy.
Config files are loaded from /etc/clickhouse-server/config.xml and the config.d/ directory. Files in config.d/ override/merge with the main config. The <replace="replace"> attribute replaces sections entirely; without it, sections merge. User-level settings can override server defaults per-session.
Q: What settings would you recommend for a production ClickHouse cluster?

**max_memory_usage: Set per-user to prevent single queries from consuming all RAM.**
max_execution_time: Prevent runaway queries.
max_threads: Control parallelism.
background_pool_size: Control merge thread count.
Set up proper RBAC with least-privilege access.
Enable log_queries = 1 for query audit trail.
Configure monitoring via system.metrics, Prometheus exporter, or Grafana.

**Category D: Docker & Deployment**
**Q: What are common Docker deployment pitfalls for ClickHouse?**

Not persisting data volumes (/var/lib/clickhouse must be a named volume or bind mount).
Not exposing the right ports (8123 for HTTP, 9000 for native, 9181 for Keeper, 9234 for Raft).
Configuration file placement — config.d/ for server, users.d/ for user overrides.
Docker networking — containers on different networks can't communicate.
Resource limits — Docker memory limits can cause OOM kills that look like crashes.

Q: How do you upgrade ClickHouse in a clustered environment?
Rolling upgrade: one node at a time, verify replication health after each. Check release notes for breaking changes. Always backup Keeper data before upgrades. Test the new version in staging first. Never skip major versions — upgrade incrementally.
Category E: Support Process & Customer Communication
Q: How do you handle an angry customer whose cluster is down?

Acknowledge urgency — "I understand this is critical and I'm prioritizing this right now."
Gather essential info quickly — version, error logs, recent changes, cluster topology.
Provide immediate mitigation if possible before root cause analysis.
Communicate regularly — even if you don't have a fix, update the customer on progress.
Document everything for the post-mortem.

**Q: When do you escalate a support case?**

When you've exhausted your troubleshooting knowledge.
When the issue involves core engine bugs requiring code-level investigation.
When the customer's SLA deadline is approaching.
When the issue affects multiple customers (potential product-wide bug).
When the customer explicitly requests escalation.
**
Q: How do you write a good support response?**

Start with acknowledging the issue.
Explain the root cause clearly (not just the fix).
Provide step-by-step instructions the customer can follow.
Include verification steps so they can confirm the fix works.
Offer proactive best practices to prevent similar issues.
End with an open offer for follow-up questions.


**Part 3: Live SQL Exercise (15 min)**
You'll work with these two tables:
support_engineers: name (String), id (UInt8)
- Sante (2), Konsta (3), Camilo (1), Derek (4), Thom (5)

cases_assigned: case_number (Nullable(String)), assignee_id (UInt8), date (Date)
- Camilo (id=1): 3 cases
- Sante (id=2): 2 cases
- Konsta (id=3): 6 cases
- Derek (id=4): 1 case
- Thom (id=5): 0 cases
**Likely SQL Questions and Answers**
1. List all cases with the engineer's name (basic JOIN)
sqlSELECT
    se.name,
    ca.case_number,
    ca.date
FROM cases_assigned AS ca
INNER JOIN support_engineers AS se ON ca.assignee_id = se.id
ORDER BY ca.date;
2. Show all engineers, including those with no cases (LEFT JOIN)
sqlSELECT
    se.name,
    ca.case_number,
    ca.date
FROM support_engineers AS se
LEFT JOIN cases_assigned AS ca ON se.id = ca.assignee_id
ORDER BY se.name;
Note: Thom (id=5) will appear with NULL case_number and date.
3. Count cases per engineer
sqlSELECT
    se.name,
    count(ca.case_number) AS case_count
FROM support_engineers AS se
LEFT JOIN cases_assigned AS ca ON se.id = ca.assignee_id
GROUP BY se.name
ORDER BY case_count DESC;
Expected: Konsta=6, Camilo=3, Sante=2, Derek=1, Thom=0
4. Find engineers with no cases assigned
sqlSELECT se.name
FROM support_engineers AS se
LEFT JOIN cases_assigned AS ca ON se.id = ca.assignee_id
WHERE ca.case_number IS NULL;
Answer: Thom
Alternative with NOT EXISTS:
sqlSELECT name
FROM support_engineers AS se
WHERE NOT EXISTS (
    SELECT 1 FROM cases_assigned WHERE assignee_id = se.id
);
5. Find the engineer with the most cases
sqlSELECT
    se.name,
    count() AS case_count
FROM cases_assigned AS ca
JOIN support_engineers AS se ON ca.assignee_id = se.id
GROUP BY se.name
ORDER BY case_count DESC
LIMIT 1;
Answer: Konsta with 6 cases.
6. Cases per day (date-based aggregation)
sqlSELECT
    date,
    count() AS cases_per_day
FROM cases_assigned
GROUP BY date
ORDER BY date;
7. Find days where more than one case was assigned
sqlSELECT
    date,
    count() AS case_count
FROM cases_assigned
GROUP BY date
HAVING case_count > 1
ORDER BY date;
8. Average cases per engineer
sqlSELECT
    avg(case_count) AS avg_cases
FROM (
    SELECT
        assignee_id,
        count() AS case_count
    FROM cases_assigned
    GROUP BY assignee_id
);
9. Date range of cases per engineer
sqlSELECT
    se.name,
    min(ca.date) AS first_case,
    max(ca.date) AS last_case,
    max(ca.date) - min(ca.date) AS days_span
FROM cases_assigned AS ca
JOIN support_engineers AS se ON ca.assignee_id = se.id
GROUP BY se.name;
10. Rank engineers by case count (window function)
sqlSELECT
    se.name,
    count() AS case_count,
    rank() OVER (ORDER BY count() DESC) AS rnk
FROM cases_assigned AS ca
JOIN support_engineers AS se ON ca.assignee_id = se.id
GROUP BY se.name;
11. Running total of cases by date
sqlSELECT
    date,
    count() AS daily_cases,
    sum(count()) OVER (ORDER BY date) AS running_total
FROM cases_assigned
GROUP BY date
ORDER BY date;
12. Find duplicate case numbers (data quality check)
sqlSELECT
    case_number,
    count() AS cnt
FROM cases_assigned
GROUP BY case_number
HAVING cnt > 1;
13. Engineers who had cases on consecutive days
sqlSELECT DISTINCT se.name
FROM cases_assigned AS ca1
JOIN cases_assigned AS ca2
    ON ca1.assignee_id = ca2.assignee_id
    AND ca2.date = ca1.date + 1
JOIN support_engineers AS se ON ca1.assignee_id = se.id;
14. Pivoting — cases per engineer per week
sqlSELECT
    se.name,
    toStartOfWeek(ca.date) AS week_start,
    count() AS cases
FROM cases_assigned AS ca
JOIN support_engineers AS se ON ca.assignee_id = se.id
GROUP BY se.name, week_start
ORDER BY week_start, se.name;
ClickHouse-Specific SQL Features to Know

count() — ClickHouse allows count() without arguments (equivalent to count(*)).
toStartOfWeek(), toStartOfMonth(), toYYYYMM() — date functions.
formatDateTime(date, '%Y-%m-%d') — date formatting.
Nullable — the case_number column is Nullable(String), so be aware of NULL handling.
ifNull(x, default) — handle NULLs.
arrayJoin — unique to ClickHouse, unfolds arrays into rows.
WITH clause — ClickHouse supports CTEs.
ClickHouse JOINs: default is ALL join (not ANY). Be explicit about join strictness if needed.


Part 4: Additional Scenarios the Team Might Throw at You
Scenario 1: Replication Lag
"Our replica is falling behind. Queries on the replica return stale data."

Check system.replicas — look at absolute_delay, queue_size, inserts_in_queue.
Check Keeper health — replication log is stored there.
Check network between nodes.
Check if the replica's merge threads are saturated.
Temporary fix: SYSTEM SYNC REPLICA tablename.

Scenario 2: Disk Space Running Out
"Our ClickHouse server is running out of disk."

Check system.parts — identify large tables/partitions.
Check system.detached_parts — old detached parts may be lingering.
Check system.mutations — stuck mutations keep old parts alive.
Use OPTIMIZE TABLE ... FINAL to force merges and remove old parts.
Set up TTL to automatically drop old data.
Consider tiered storage (hot/cold with storage policies).

**Scenario 3: Query Timeout**
"Our dashboard queries keep timing out."

**Check max_execution_time setting.**
Look at the query plan — is a full table scan happening?
Is the ORDER BY key aligned with the query's WHERE clause?
Consider pre-aggregation with materialized views.
Check concurrent query load via system.processes.

**Scenario 4: Authentication Issues**
"We set up LDAP auth and users can't log in."

**Check ClickHouse logs for LDAP connection errors.**
Verify LDAP server is reachable from the ClickHouse host.
Check <ldap_servers> config in config.xml.
Verify user mapping — does the ClickHouse user exist with LDAP auth?
Test LDAP connectivity with ldapsearch from the ClickHouse host.

**Scenario 5: Cluster Configuration**
"We want to add a new shard to our existing cluster."

Update the <remote_servers> config on all nodes.
The new shard won't have existing data — need to redistribute.
For ReplicatedMergeTree, ensure Keeper paths are correct for the new shard.
Consider using Distributed table engine to route queries across shards.
Rebalancing existing data requires manual partition moves or re-insertion.


**Part 5: Questions to Ask the Interviewers (10 min)**
Having thoughtful questions shows genuine interest. Here are strong ones:

"What does a typical day look like for a support engineer on this team? How are cases prioritized and assigned?"
"What's the most common category of issues customers bring to the support team?"
"How does the support team collaborate with the engineering team when a bug is identified?"
"What tools does the team use for case management and internal knowledge sharing?"
"How do you handle on-call rotations? What's the expected response time for critical issues?"
"What does the ramp-up process look like for new support engineers? Is there a mentorship program?"
"How does the team stay current with new ClickHouse features and releases?"
"What's the ratio of ClickHouse Cloud vs. self-hosted customer issues?"

**
**Part 6: Quick Reference — Key ClickHouse Concepts to Have Ready****
ConceptOne-linerColumnar storageData stored by column, not row — great for analytics, bad for point lookupsSparse indexOne entry per 8192 rows (granule), fits in RAM, skips irrelevant granulesMergeTreeCore engine: sorted parts + background mergesReplicatedMergeTreeMergeTree + replication via KeeperKeeperClickHouse's built-in ZooKeeper replacement using Raft consensusRaft consensusMajority-based agreement protocol; needs odd number of nodesDistributed tableVirtual table that fans queries out to shardsMaterialized viewINSERT trigger that transforms and writes to a target tableSettings profileNamed bundle of per-query settings (memory, time limits)Row policyTransparent row-level filter applied to a user/roleTTLAuto-expire or move data based on a date columnMutationsALTER TABLE ... UPDATE/DELETE — async, heavy operationsPartsPhysical data files on disk; too many = performance problemProjectionsPre-sorted/pre-aggregated copies within a table for faster queries

**Final Tips for Interview Day**

Have your Docker environment ready with both tables loaded and ClickHouse running. Test it before the interview.
Have your assignment open — you'll present from it. Know it inside out.
Think out loud — they're evaluating your process, not just your answers. Narrate your reasoning.
It's OK to say "I'd need to check the docs" — good support engineers know when to look things up vs. guess.
Be customer-friendly in tone — remember the case review is a role-play where they are the customer.
Practice the SQL exercise by running the actual queries beforehand so you're comfortable with ClickHouse's SQL dialect.
