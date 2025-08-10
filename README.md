# PostgreSQL Partitioning for Zabbix `events` Table

## 📌 Overview
This migration converts the `events` table in Zabbix's PostgreSQL database into a **range-partitioned table** based on `eventid`.  
Partitioning improves query performance, reduces table bloat, and makes cleanup/archival faster.

## 🛠 Steps Performed

### 1. Create Partitioned Table
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

### 2. Create Partitions
```sql
CREATE TABLE events_p1 PARTITION OF events_new FOR VALUES FROM (0) TO (100000000);
CREATE TABLE events_p2 PARTITION OF events_new FOR VALUES FROM (100000000) TO (200000000);
CREATE TABLE events_p3 PARTITION OF events_new FOR VALUES FROM (200000000) TO (300000000);
```

### 3. Migrate Data
```sql
INSERT INTO events_new SELECT * FROM events;
```

### 4. Swap Old and New Tables
```sql
ALTER TABLE events RENAME TO events_old;
ALTER TABLE events_new RENAME TO events;
```

### 5. Recreate Indexes
```sql
CREATE INDEX idx_events_clock ON events (clock);
CREATE INDEX idx_events_objectid ON events (objectid);
```

### 6. Add Primary Key
```sql
ALTER TABLE events ADD PRIMARY KEY (eventid);
```

### 7. Update Foreign Keys
Example:
```sql
ALTER TABLE event_tag DROP CONSTRAINT c_event_tag_1;
ALTER TABLE event_tag
  ADD CONSTRAINT c_event_tag_1 FOREIGN KEY (eventid)
  REFERENCES events (eventid) ON DELETE CASCADE;
```
Repeat for:
- `problem` (`c_problem_1`, `c_problem_2`)
- `alerts` (`c_alerts_2`, `c_alerts_5`)
- `acknowledges` (`c_acknowledges_2`)
- `event_recovery` (`c_event_recovery_1`, `c_event_recovery_2`, `c_event_recovery_3`)
- `event_suppress` (`c_event_suppress_1`)

---

## ✅ Verification
After migration, run:
```sql
-- Check partitions
SELECT inhrelid::regclass AS partition_name
FROM pg_inherits
WHERE inhparent = 'events'::regclass;

-- Count rows
SELECT relname, n_live_tup
FROM pg_stat_user_tables
WHERE relname LIKE 'events%';
```

---

## ⚠️ Notes & Warnings
- Perform this migration during a maintenance window — **no inserts/updates** to `events` should happen during the copy step.
- Test on staging before production.
- Keep `events_old` until you verify the migration.
- Dropping/re-adding foreign keys is **mandatory** so child tables point to the new partitioned `events`.

---
readme_auto_partition = """# Auto Partition Creator for Zabbix Events Table

## 📌 Overview
This script checks the `events` table in a PostgreSQL Zabbix database and **automatically creates a new partition** when the current maximum `eventid` reaches the end of an existing range.

Partitioning helps improve query performance and makes old data management easier.

---

## 🛠 Script Details

**File:** `auto_partition.sh`

### Bash Script
```bash
#!/bin/bash
# Auto-create new partition for Zabbix events table if needed

DB_NAME="zabbix"
DB_USER="postgres"
PARTITION_SIZE=100000000  # size of each range
PARENT_TABLE="events"

# Get current max eventid
MAX_EVENTID=$(psql -U "$DB_USER" -d "$DB_NAME" -t -c "SELECT COALESCE(MAX(eventid),0) FROM $PARENT_TABLE;" | tr -d '[:space:]')

# Calculate next partition range
NEXT_START=$(( (MAX_EVENTID / PARTITION_SIZE) * PARTITION_SIZE ))
NEXT_END=$(( NEXT_START + PARTITION_SIZE ))

# Check if partition already exists
PARTITION_NAME="${PARENT_TABLE}_p$((NEXT_START / PARTITION_SIZE + 1))"
EXISTS=$(psql -U "$DB_USER" -d "$DB_NAME" -t -c "SELECT to_regclass('$PARTITION_NAME');" | tr -d '[:space:]')

if [ "$EXISTS" = "" ] || [ "$EXISTS" = "null" ]; then
    echo "Creating new partition: $PARTITION_NAME for range $NEXT_START to $NEXT_END..."
    psql -U "$DB_USER" -d "$DB_NAME" -c "CREATE TABLE $PARTITION_NAME PARTITION OF $PARENT_TABLE FOR VALUES FROM ($NEXT_START) TO ($NEXT_END);"
else
    echo "Partition $PARTITION_NAME already exists. No action taken."
fi

---
## 🗑 Optional/Debug Commands
These were useful during development but are **not required** for migration:
```sql
SELECT COUNT(*) FROM events;
SELECT MAX(eventid), clock FROM events;
SELECT column_name, column_default
FROM information_schema.columns
WHERE table_name IN ('events_p1', 'events_p2', 'events_p3')
  AND column_name = 'acknowledged';
```
