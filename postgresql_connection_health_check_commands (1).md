# PostgreSQL Connection Health Check Commands

These are the **day-to-day DBA commands** I would keep as a connection-management/SOP cheat sheet.

## 1. Basic connection health

### Current database/user/session

```
```

```
SELECT
    pg_backend_pid() AS pid,
    current_user,
    current_database(),
    inet_server_addr() AS server_ip,
    inet_server_port() AS server_port;
```

### PostgreSQL version

```
```

```
SELECT version();
```

### Current connection count

```
```

```
SELECT count(*) AS current_connections
FROM pg_stat_activity;
```

### Current connections vs `max_connections`

```
```

```
SELECT
    count(*) AS current_connections,
    current_setting('max_connections')::int AS max_connections,
    current_setting('superuser_reserved_connections')::int
        AS superuser_reserved_connections
FROM pg_stat_activity;
```

---

# 2. Connection utilization

### Connection usage %

```
```

```
SELECT
    count(*) AS current_connections,
    current_setting('max_connections')::int AS max_connections,
    round(
        100.0 * count(*) /
        current_setting('max_connections')::int,
        2
    ) AS usage_percent
FROM pg_stat_activity;
```

### Available connection slots

```
```

```
SELECT
    current_setting('max_connections')::int
        - count(*) AS available_slots
FROM pg_stat_activity;
```

### Check configuration

```
```

```
SHOW max_connections;

SHOW superuser_reserved_connections;
```

---

# 3. All sessions

### Complete `pg_stat_activity` view

```
```

```
SELECT
    pid,
    usename,
    datname,
    client_addr,
    client_hostname,
    client_port,
    application_name,
    backend_start,
    xact_start,
    query_start,
    state_change,
    state,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
ORDER BY backend_start;
```

### More DBA-friendly version

```
```

```
SELECT
    pid,
    usename,
    datname,
    client_addr,
    application_name,
    state,
    now() - backend_start AS connection_age,
    now() - xact_start AS transaction_age,
    now() - query_start AS query_age,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
ORDER BY backend_start;
```

---

# 4. Sessions by state

### Active / idle / idle in transaction

```
```

```
SELECT
    state,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY state
ORDER BY state;
```

### Only client sessions

```
```

```
SELECT
    state,
    count(*) AS connections
FROM pg_stat_activity
WHERE backend_type = 'client backend'
GROUP BY state
ORDER BY state;
```

This avoids mixing normal client connections with PostgreSQL background processes.

---

# 5. Active sessions

### All active queries

```
```

```
SELECT
    pid,
    usename,
    datname,
    client_addr,
    application_name,
    query_start,
    now() - query_start AS query_duration,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY query_start;
```

### Queries running longer than 1 minute

```
```

```
SELECT
    pid,
    usename,
    datname,
    application_name,
    query_start,
    now() - query_start AS query_duration,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE state = 'active'
  AND now() - query_start > interval '1 minute'
ORDER BY query_start;
```

### Queries running longer than 5 minutes

```
```

```
SELECT
    pid,
    usename,
    datname,
    application_name,
    now() - query_start AS query_duration,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE state = 'active'
  AND now() - query_start > interval '5 minutes'
ORDER BY query_start;
```

---

# 6. Idle sessions

### All idle sessions

```
```

```
SELECT
    pid,
    usename,
    datname,
    client_addr,
    application_name,
    state_change,
    now() - state_change AS idle_duration
FROM pg_stat_activity
WHERE state = 'idle'
ORDER BY state_change;
```

### Idle for more than 5 minutes

```
```

```
SELECT
    pid,
    usename,
    datname,
    client_addr,
    application_name,
    state_change,
    now() - state_change AS idle_duration
FROM pg_stat_activity
WHERE state = 'idle'
  AND now() - state_change > interval '5 minutes'
ORDER BY state_change;
```

### Longest idle sessions

```
```

```
SELECT
    pid,
    usename,
    datname,
    application_name,
    now() - state_change AS idle_duration
FROM pg_stat_activity
WHERE state = 'idle'
ORDER BY state_change
LIMIT 20;
```

---

# 7. Idle in transaction — very important

### Find all

```
```

```
SELECT
    pid,
    usename,
    datname,
    client_addr,
    application_name,
    xact_start,
    state_change,
    now() - xact_start AS transaction_age,
    query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY xact_start;
```

### Older than 5 minutes

```
```

```
SELECT
    pid,
    usename,
    datname,
    application_name,
    xact_start,
    now() - xact_start AS transaction_age,
    query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND now() - xact_start > interval '5 minutes'
ORDER BY xact_start;
```

### Oldest idle transactions

```
```

```
SELECT
    pid,
    usename,
    datname,
    application_name,
    now() - xact_start AS transaction_age,
    query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY xact_start
LIMIT 20;
```

---

# 8. Connection count by database

```
```

```
SELECT
    datname,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY datname
ORDER BY connections DESC;
```

---

# 9. Connection count by user

```
```

```
SELECT
    usename,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY usename
ORDER BY connections DESC;
```

---

# 10. Connection count by application

```
```

```
SELECT
    application_name,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY application_name
ORDER BY connections DESC;
```

### Application + state

```
```

```
SELECT
    application_name,
    state,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY application_name, state
ORDER BY application_name, state;
```

This is particularly useful for your Python lab.

```
```

```
SELECT
    application_name,
    state,
    count(*) AS connections
FROM pg_stat_activity
WHERE application_name LIKE 'python_%'
GROUP BY application_name, state
ORDER BY application_name, state;
```

---

# 11. Connections by client IP

```
```

```
SELECT
    client_addr,
    count(*) AS connections
FROM pg_stat_activity
WHERE client_addr IS NOT NULL
GROUP BY client_addr
ORDER BY connections DESC;
```

### Client IP + application

```
```

```
SELECT
    client_addr,
    application_name,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY client_addr, application_name
ORDER BY count(*) DESC;
```

---

# 12. Oldest connections

```
```

```
SELECT
    pid,
    usename,
    datname,
    client_addr,
    application_name,
    backend_start,
    now() - backend_start AS connection_age,
    state
FROM pg_stat_activity
ORDER BY backend_start
LIMIT 20;
```

---

# 13. Waiting sessions

### Sessions waiting for something

```
```

```
SELECT
    pid,
    usename,
    datname,
    application_name,
    state,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
ORDER BY pid;
```

### Group waits

```
```

```
SELECT
    wait_event_type,
    wait_event,
    count(*) AS sessions
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
GROUP BY wait_event_type, wait_event
ORDER BY sessions DESC;
```

---

# 14. Blocking sessions

### Find blocked queries

```
```

```
SELECT
    pid,
    usename,
    datname,
    application_name,
    query_start,
    now() - query_start AS waiting_duration,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock'
ORDER BY query_start;
```

### Blocked → blocking PID

```
```

```
SELECT
    blocked.pid AS blocked_pid,
    blocked.usename AS blocked_user,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.usename AS blocking_user,
    blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
    ON blocking.pid = ANY(
        pg_blocking_pids(blocked.pid)
    );
```

### Better blocking report

```
```

```
SELECT
    blocked.pid AS blocked_pid,
    blocked.application_name AS blocked_app,
    now() - blocked.query_start AS blocked_duration,
    blocked.query AS blocked_query,

    blocking.pid AS blocking_pid,
    blocking.application_name AS blocking_app,
    blocking.state AS blocking_state,
    now() - blocking.xact_start AS blocking_xact_age,
    blocking.query AS blocking_query

FROM pg_stat_activity blocked
CROSS JOIN LATERAL unnest(
    pg_blocking_pids(blocked.pid)
) AS blocker_pid
JOIN pg_stat_activity blocking
    ON blocking.pid = blocker_pid
WHERE blocked.wait_event_type = 'Lock';
```

---

# 15. Cancel a running query

Use this when the **query** is the problem but you want to preserve the session.

```
```

```
SELECT pg_cancel_backend(<PID>);
```

Example:

```
```

```
SELECT pg_cancel_backend(12345);
```

Verify:

```
```

```
SELECT
    pid,
    state,
    query
FROM pg_stat_activity
WHERE pid = 12345;
```

---

# 16. Terminate a session

Use this when you need to disconnect the client session.

```
```

```
SELECT pg_terminate_backend(<PID>);
```

Example:

```
```

```
SELECT pg_terminate_backend(12345);
```

**DBA rule:**

```
```

```
pg_cancel_backend()
        ↓
Cancel query

pg_terminate_backend()
        ↓
Terminate session
```

---

# 17. Don't kill your own session

Very important when generating termination commands.

```
```

```
SELECT
    pid,
    usename,
    datname,
    application_name,
    client_addr,
    state
FROM pg_stat_activity
WHERE pid <> pg_backend_pid();
```

---

# 18. Generate commands to terminate idle sessions

Don't blindly execute them. First inspect the generated commands.

```
```

```
SELECT
    format(
        'SELECT pg_terminate_backend(%s);',
        pid
    ) AS terminate_command
FROM pg_stat_activity
WHERE state = 'idle'
  AND pid <> pg_backend_pid();
```

### Idle sessions older than 30 minutes

```
```

```
SELECT
    format(
        'SELECT pg_terminate_backend(%s);',
        pid
    ) AS terminate_command
FROM pg_stat_activity
WHERE state = 'idle'
  AND now() - state_change > interval '30 minutes'
  AND pid <> pg_backend_pid();
```

---

# 19. Generate commands for idle-in-transaction

```
```

```
SELECT
    format(
        'SELECT pg_terminate_backend(%s);',
        pid
    ) AS terminate_command
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND pid <> pg_backend_pid();
```

For example, only older than 10 minutes:

```
```

```
SELECT
    format(
        'SELECT pg_terminate_backend(%s);',
        pid
    ) AS terminate_command
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND now() - xact_start > interval '10 minutes'
  AND pid <> pg_backend_pid();
```

---

# 20. Application-specific connections

For your Python lab:

```
```

```
SELECT
    pid,
    usename,
    datname,
    application_name,
    client_addr,
    state,
    backend_start,
    xact_start,
    query_start,
    state_change,
    query
FROM pg_stat_activity
WHERE application_name LIKE 'python_%'
ORDER BY application_name;
```

### Count them

```
```

```
SELECT count(*)
FROM pg_stat_activity
WHERE application_name LIKE 'python_%';
```

### State distribution

```
```

```
SELECT
    state,
    count(*)
FROM pg_stat_activity
WHERE application_name LIKE 'python_%'
GROUP BY state
ORDER BY state;
```

---

# 21. Connection-related timeout health check

Check all important timeout settings:

```
```

```
SHOW statement_timeout;

SHOW lock_timeout;

SHOW idle_in_transaction_session_timeout;

SHOW idle_session_timeout;

SHOW transaction_timeout;
```

Or in one query:

```
```

```
SELECT
    name,
    setting,
    unit,
    context
FROM pg_settings
WHERE name IN (
    'statement_timeout',
    'lock_timeout',
    'idle_in_transaction_session_timeout',
    'idle_session_timeout',
    'transaction_timeout'
)
ORDER BY name;
```

---

# 22. Connection logging

Check whether connection logging is enabled:

```
```

```
SHOW log_connections;

SHOW log_disconnections;
```

You can also check:

```
```

```
SELECT
    name,
    setting
FROM pg_settings
WHERE name IN (
    'log_connections',
    'log_disconnections',
    'log_duration'
);
```

---

# 23. Database-level connection statistics

```
```

```
SELECT
    datname,
    numbackends,
    sessions,
    sessions_abandoned,
    sessions_fatal,
    sessions_killed
FROM pg_stat_database
ORDER BY numbackends DESC;
```

If you're working across PostgreSQL versions, check the available columns on your version before using the complete query.

---

# 24. Connection health — one quick DBA query

When someone says:

> **"DB connections are high. Check it."**

Start with:

```
```

```
SELECT
    state,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY state
ORDER BY connections DESC;
```

Then:

```
```

```
SELECT
    datname,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY datname
ORDER BY connections DESC;
```

Then:

```
```

```
SELECT
    usename,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY usename
ORDER BY connections DESC;
```

Then:

```
```

```
SELECT
    application_name,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY application_name
ORDER BY connections DESC;
```

Then:

```
```

```
SELECT
    count(*) AS current_connections,
    current_setting('max_connections')::int AS max_connections,
    round(
        100.0 * count(*) /
        current_setting('max_connections')::int,
        2
    ) AS usage_percent
FROM pg_stat_activity;
```

---

# 25. Your real-time connection-exhaustion lab

While your Python script is running, I would keep these **four queries** open.

### A. Overall

```
```

```
SELECT
    count(*) AS current_connections,
    current_setting('max_connections')::int AS max_connections,
    round(
        100.0 * count(*) /
        current_setting('max_connections')::int,
        2
    ) AS usage_percent
FROM pg_stat_activity;
```

### B. State distribution

```
```

```
SELECT
    state,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY state
ORDER BY state;
```

### C. Python sessions

```
```

```
SELECT
    application_name,
    state,
    count(*) AS connections
FROM pg_stat_activity
WHERE application_name LIKE 'python_%'
GROUP BY application_name, state
ORDER BY application_name, state;
```

### D. Long idle-in-transaction

```
```

```
SELECT
    pid,
    usename,
    application_name,
    now() - xact_start AS transaction_age,
    query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY xact_start;
```

That gives you a very natural DBA troubleshooting flow:

```
```

```
CONNECTION PROBLEM
       ↓
How many connections?
       ↓
What % of max_connections?
       ↓
Which state?
       ↓
Which database?
       ↓
Which user?
       ↓
Which application?
       ↓
Which client IP?
       ↓
Any long idle sessions?
       ↓
Any idle-in-transaction?
       ↓
Any long-running active queries?
       ↓
Any blocked sessions?
       ↓
Cancel query / terminate session
       ↓
Why did connection exhaustion happen?
       ↓
PgBouncer
```

This is the **connection-healthcheck section** I'd put directly into your DBA SOP before the PgBouncer section.


---

# 26. Terminate all idle sessions except your own

Use this when you need to quickly clear **idle** client connections while making sure the current DBA session is not terminated.

> **Important:** Review the sessions first. Terminating idle sessions disconnects their clients and may affect applications that expect persistent connections.

### Generate termination commands

```sql
SELECT
    format('SELECT pg_terminate_backend(%s);', pid) AS terminate_command
FROM pg_stat_activity
WHERE state = 'idle'
  AND pid <> pg_backend_pid();
```

### Execute directly

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle'
  AND pid <> pg_backend_pid();
```

This terminates **only sessions in `idle` state** and excludes the current session.

---

# 27. Terminate all idle-in-transaction sessions except your own

Use this when connections are stuck in **`idle in transaction`** and you need to release those sessions quickly.

> **Important:** Terminating an `idle in transaction` session will roll back its open transaction. Review the sessions before executing this in production.

### Generate termination commands

```sql
SELECT
    format('SELECT pg_terminate_backend(%s);', pid) AS terminate_command
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND pid <> pg_backend_pid();
```

### Execute directly

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND pid <> pg_backend_pid();
```

This terminates **only `idle in transaction` sessions** and excludes the current session.

---

## Quick emergency cleanup

```text
Idle sessions
     ↓
SELECT pg_terminate_backend(pid)
WHERE state = 'idle'
AND pid <> pg_backend_pid();

Idle in transaction
     ↓
SELECT pg_terminate_backend(pid)
WHERE state = 'idle in transaction'
AND pid <> pg_backend_pid();
```

**Do not use these commands blindly in production.** First identify the application, user, transaction age, and business impact of the sessions you are about to terminate.
