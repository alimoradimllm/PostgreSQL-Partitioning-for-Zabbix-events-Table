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

```bash
#!/bin/bash

DB_NAME="zabbix"
DB_USER="zabbix"

# Get latest partition name
latest_partition=$(psql -U "$DB_USER" -d "$DB_NAME" -t -c "
    SELECT tablename 
    FROM pg_tables 
    WHERE tablename LIKE 'events_%' 
    ORDER BY tablename DESC LIMIT 1;
" | xargs)

# Extract year and month
year_month=$(echo "$latest_partition" | sed -E 's/events_([0-9]{4})_([0-9]{2})/\1-\2/')
next_month=$(date -d "$year_month-01 +1 month" +"%Y-%m")

next_year=$(echo "$next_month" | cut -d- -f1)
next_month_num=$(echo "$next_month" | cut -d- -f2)

# Create next partition
psql -U "$DB_USER" -d "$DB_NAME" -c "
CREATE TABLE events_${next_year}_${next_month_num} PARTITION OF events
FOR VALUES FROM (EXTRACT(EPOCH FROM TIMESTAMP '${next_year}-${next_month_num}-01 00:00:00')::BIGINT)
TO (EXTRACT(EPOCH FROM TIMESTAMP '$(date -d "$next_month-01 +1 month" +"%Y-%m-%d") 00:00:00')::BIGINT);
"
```

💡 **Password Handling**  
To avoid entering the DB password every time:  
```bash
export PGPASSWORD="your_password"
```
Or configure `~/.pgpass` for secure storage.

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
