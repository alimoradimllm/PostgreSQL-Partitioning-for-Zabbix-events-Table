
# PostgreSQL Range Partitioning for `events` Table

This guide explains how to migrate the `events` table in Zabbix's PostgreSQL database to use **range partitioning** by `eventid`.  
Partitioning improves query performance, reduces table bloat, and speeds up cleanup and archival.

---

## 1️⃣ Migration Steps

### Step 1 – Backup the Table
```bash
pg_dump -U zabbix -t events zabbix > events_backup.sql
```

### Step 2 – Create Partitioned Table
```sql
CREATE TABLE events_new (
    eventid BIGINT NOT NULL,
    source INTEGER NOT NULL,
    object INTEGER NOT NULL,
    objectid BIGINT NOT NULL,
    clock INTEGER NOT NULL,
    value INTEGER NOT NULL,
    acknowledged INTEGER NOT NULL,
    ns INTEGER NOT NULL,
    name VARCHAR(255),
    severity INTEGER DEFAULT 0
) PARTITION BY RANGE (eventid);
```

### Step 3 – Create Partitions
```sql
CREATE TABLE events_p1 PARTITION OF events_new FOR VALUES FROM (0) TO (100000000);
CREATE TABLE events_p2 PARTITION OF events_new FOR VALUES FROM (100000000) TO (200000000);
CREATE TABLE events_p3 PARTITION OF events_new FOR VALUES FROM (200000000) TO (300000000);
```

### Step 4 – Migrate Data
```sql
INSERT INTO events_new SELECT * FROM events;
```

### Step 5 – Swap Old and New Tables
```sql
ALTER TABLE events RENAME TO events_old;
ALTER TABLE events_new RENAME TO events;
```

### Step 6 – Recreate Indexes
```sql
CREATE INDEX idx_events_clock ON events (clock);
CREATE INDEX idx_events_objectid ON events (objectid);
```

### Step 7 – Add Primary Key
```sql
ALTER TABLE events ADD PRIMARY KEY (eventid);
```

### Step 8 – Update Foreign Keys
Example for `event_tag` table:
```sql
ALTER TABLE event_tag DROP CONSTRAINT c_event_tag_1;

ALTER TABLE event_tag
  ADD CONSTRAINT c_event_tag_1 FOREIGN KEY (eventid)
  REFERENCES events (eventid) ON DELETE CASCADE;
```
Repeat similar steps for other tables and constraints:
- `problem` (`c_problem_1`, `c_problem_2`)
- `alerts` (`c_alerts_2`, `c_alerts_5`)
- `acknowledges` (`c_acknowledges_2`)
- `event_recovery` (`c_event_recovery_1`, `c_event_recovery_2`, `c_event_recovery_3`)
- `event_suppress` (`c_event_suppress_1`)

---

## ✅ Verification

Check created partitions:
```sql
SELECT inhrelid::regclass AS partition_name
FROM pg_inherits
WHERE inhparent = 'events'::regclass;
```

Check row counts:
```sql
SELECT relname, n_live_tup
FROM pg_stat_user_tables
WHERE relname LIKE 'events%';
```

---

## 2️⃣ Auto-Partition Script (`auto_partition.sh`)

PostgreSQL does **not** auto-create partitions when ranges fill up.  
Use this Bash script to check the highest `eventid` and create new partitions automatically.

```bash
#!/bin/bash
# Auto-create new partition for Zabbix events table if needed

export PGPASSWORD="your_password"

DB_NAME="zabbix"
DB_USER="postgres"
PARTITION_SIZE=100000000  # size of each range
PARENT_TABLE="events"

# Get current max eventid
MAX_EVENTID=$(psql -U "$DB_USER" -d "$DB_NAME" -t -c "SELECT COALESCE(MAX(eventid),0) FROM $PARENT_TABLE;" | tr -d '[:space:]')

# Calculate next partition range
NEXT_START=$(( (MAX_EVENTID / PARTITION_SIZE) * PARTITION_SIZE ))
NEXT_END=$(( NEXT_START + PARTITION_SIZE ))

# Determine partition name
PARTITION_NAME="${PARENT_TABLE}_p$((NEXT_START / PARTITION_SIZE + 1))"

# Check if partition exists
EXISTS=$(psql -U "$DB_USER" -d "$DB_NAME" -t -c "SELECT to_regclass('$PARTITION_NAME');" | tr -d '[:space:]')

if [ "$EXISTS" = "" ] || [ "$EXISTS" = "null" ]; then
    echo "Creating new partition: $PARTITION_NAME for range $NEXT_START to $NEXT_END..."
    psql -U "$DB_USER" -d "$DB_NAME" -c "CREATE TABLE $PARTITION_NAME PARTITION OF $PARENT_TABLE FOR VALUES FROM ($NEXT_START) TO ($NEXT_END);"
else
    echo "Partition $PARTITION_NAME already exists. No action taken."
fi

unset PGPASSWORD
```

---

## ⚠️ Notes & Warnings

- Perform this migration during a maintenance window; **no inserts/updates** to `events` should occur during data migration.
- Always test on a staging environment before production.
- Keep the `events_old` table until the migration is verified.
- Dropping and re-adding foreign keys is **mandatory** to ensure child tables reference the new partitioned `events`.
- Adjust `PGPASSWORD` handling for your security policies.

---

## 3️⃣ Optional / Debug Commands

List all `events` partitions:
```bash
\dt events_*
```

View partition details:
```sql
SELECT * FROM pg_partitions WHERE tablename LIKE 'events_%';
```

Count rows in main and partitions:
```sql
SELECT COUNT(*) FROM events;
SELECT COUNT(*) FROM events_p1;
SELECT COUNT(*) FROM events_p2;
SELECT COUNT(*) FROM events_p3;
```

---

*End of Guide*
