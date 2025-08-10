# PostgreSQL Range Partitioning for `events` Table

This guide explains how to migrate the `events` table to **PostgreSQL range partitioning** and how to automatically create new partitions using a Bash script.

---

## 1️⃣ Migration Steps

**Step 1 – Backup the table**
```bash
pg_dump -U zabbix -t events zabbix > events_backup.sql
```

**Step 2 – Create a new partitioned table**
```sql
CREATE TABLE events_new (
    eventid BIGSERIAL PRIMARY KEY,
    source INTEGER NOT NULL,
    object INTEGER NOT NULL,
    objectid BIGINT NOT NULL,
    clock INTEGER NOT NULL,
    value INTEGER NOT NULL,
    acknowledged INTEGER NOT NULL,
    ns INTEGER NOT NULL
) PARTITION BY RANGE (clock);
```

**Step 3 – Create partitions (one per month)**
```sql
CREATE TABLE events_2025_08 PARTITION OF events_new
FOR VALUES FROM (UNIX_TIMESTAMP('2025-08-01 00:00:00')) 
TO (UNIX_TIMESTAMP('2025-09-01 00:00:00'));

CREATE TABLE events_2025_09 PARTITION OF events_new
FOR VALUES FROM (UNIX_TIMESTAMP('2025-09-01 00:00:00')) 
TO (UNIX_TIMESTAMP('2025-10-01 00:00:00'));
```

**Step 4 – Copy existing data**
```sql
INSERT INTO events_new SELECT * FROM events;
```

**Step 5 – Swap the tables**
```sql
ALTER TABLE events RENAME TO events_old;
ALTER TABLE events_new RENAME TO events;
```

**Step 6 – Drop the old table (optional)**
```sql
DROP TABLE events_old;
```

---

## 2️⃣ Auto-Partition Script (`auto_partition.sh`)

This Bash script checks for the latest partition and creates the next month’s partition automatically.
PostgreSQL does not automatically create partitions when a range is full.  
This script checks the highest `eventid` and creates a new partition if needed.


```bash

### Bash Script
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

# Check if partition already exists
PARTITION_NAME="${PARENT_TABLE}_p$((NEXT_START / PARTITION_SIZE + 1))"
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

## 3️⃣ Optional / Debug Commands

Check all partitions:
```bash
\dt events_*
```

Check partition details:
```sql
SELECT * FROM pg_partitions WHERE tablename LIKE 'events_%';
```

---
