# 🦴 CAVEMAN GUIDE TO POSTGRES CDC & WAL 🦴

Me explain like we sitting around fire. No fancy words. Just truth.

---

## 🪨 PART 1: WHAT IS WAL? (Write-Ahead Log)

### Story Time 🔥

Imagine you caveman. You have **big stone tablet** (database). You carve important things on it.

But carving stone SLOW. What if you forget what to carve? What if rock fall on head while carving?

**SOLUTION:** Before carving stone, you **write on leaf first** (WAL).

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   📝 LEAF (WAL)              🪨 BIG STONE (DATABASE)    │
│   ─────────────              ────────────────────────   │
│                                                         │
│   "Add 5 fish"      ──────▶  Actually carve fish        │
│   "Remove 2 berry"  ──────▶  Actually erase berry       │
│   "Add new cave"    ──────▶  Actually carve cave        │
│                                                         │
│   FAST! SAFE!                SLOW! PERMANENT!           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Why Leaf (WAL) Good?

| Problem | How WAL Fix |
|---------|-------------|
| 🦕 Dinosaur attack mid-carving! | No worry. Look at leaf. Know what was doing. Continue. |
| 🧠 Forget what to carve? | Leaf remember everything. Leaf never forget. |
| 🐢 Stone carving slow | Write leaf fast. Carve stone later when have time. |

---

## 🪵 WHAT WAL ACTUALLY LOOK LIKE?

WAL = **List of ALL changes** in order they happen.

```
WAL FILE (like ancient scroll)
═══════════════════════════════════════════════════════

CHANGE #1 [Time: Sun up]
   Table: "food_inventory"
   Action: INSERT
   Data: {item: "mammoth_leg", quantity: 3}

───────────────────────────────────────────────────────

CHANGE #2 [Time: Sun middle]
   Table: "food_inventory"  
   Action: UPDATE
   Old: {item: "mammoth_leg", quantity: 3}
   New: {item: "mammoth_leg", quantity: 1}   ← tribe ate 2

───────────────────────────────────────────────────────

CHANGE #3 [Time: Sun down]
   Table: "cave_members"
   Action: DELETE
   Data: {name: "Grok", reason: "eaten_by_sabertooth"}

═══════════════════════════════════════════════════════
```

### In Real Postgres:

```sql
-- This INSERT...
INSERT INTO users (name, email) VALUES ('Grok', 'grok@cave.com');

-- Creates WAL record like:
-- LSN: 0/16B3D80
-- Transaction: 12345
-- Table: users (OID 16385)
-- Action: INSERT
-- Data: (tuple data in binary)
```

**LSN** = "Log Sequence Number" = **which leaf in stack** (position in WAL)

---

## 🦣 PART 2: WHAT IS CDC? (Change Data Capture)

### Simple Explain:

**CDC = Caveman who sits and WATCHES the leaf pile (WAL)**

Every time new leaf added → caveman shouts to other tribe what changed!

```
    ┌──────────────────┐
    │   POSTGRES       │
    │   DATABASE       │
    │                  │
    │  Table: foods    │───────┐
    │  Table: caves    │       │ Changes happen
    │  Table: tribe    │       │
    └──────────────────┘       ▼
                          ┌─────────┐
                          │   WAL   │ ← All changes written here
                          │ 📜📜📜  │
                          └────┬────┘
                               │
                               │ CDC WATCHES! 👀
                               ▼
                    ┌─────────────────────┐
                    │   CDC CAVEMAN       │
                    │   (Debezium/etc)    │
                    │                     │
                    │  "ME SEE CHANGE!"   │
                    │  "ME TELL OTHERS!"  │
                    └──────────┬──────────┘
                               │
           ┌───────────────────┼───────────────────┐
           ▼                   ▼                   ▼
      ┌─────────┐        ┌──────────┐        ┌──────────┐
      │ KAFKA   │        │ OTHER DB │        │ SEARCH   │
      │ 🔥      │        │ 🪨       │        │ 🔍       │
      └─────────┘        └──────────┘        └──────────┘
      
      Other tribes now know what happen in main cave!
```

---

## 🍖 WHY CDC USEFUL? REAL EXAMPLES:

### Example 1: Keep Search Cave Updated

```
MAIN DATABASE (Postgres)              SEARCH CAVE (Elasticsearch)
┌─────────────────────┐               ┌─────────────────────┐
│ products table      │               │ products index      │
│                     │    CDC        │                     │
│ INSERT iPhone ──────┼──────────────▶│ Now searchable!     │
│ UPDATE price ───────┼──────────────▶│ Price updated!      │
│ DELETE old item ────┼──────────────▶│ Removed from search!│
└─────────────────────┘               └─────────────────────┘

NO NEED write code twice! CDC do automatically!
```

### Example 2: Analytics Cave Want Data Too

```
MAIN CAVE                             ANALYTICS CAVE
(Postgres - OLTP)                     (Data Warehouse - OLAP)

Orders table                          
├─ New order #123 ─────CDC───────────▶  Raw orders table
├─ Order updated ──────CDC───────────▶  Updated in warehouse  
└─ Order cancelled ────CDC───────────▶  Marked cancelled

REAL TIME! No wait for nightly batch job!
```

### Example 3: Tell Other Microservice

```
User Service                          Email Service
┌─────────────┐                       ┌─────────────┐
│ users table │                       │             │
│             │   CDC → Kafka         │ "Oh! New    │
│ INSERT ─────┼───────────────────────▶  user! Me   │
│ new user    │                       │  send       │
│             │                       │  welcome    │
└─────────────┘                       │  email!"    │
                                      └─────────────┘
```

---

## 🦴 PART 3: HOW CDC READ WAL? (The Magic)

### Postgres Have Special Feature: **LOGICAL REPLICATION**

Normal WAL = **Physical** = Just bytes, only Postgres understand

Logical Decoding = **Translate to human/app readable!**

```
PHYSICAL WAL (gibberish):                LOGICAL WAL (understandable):
─────────────────────────                ────────────────────────────

0x4F2A881B 0x0012 0x8F21                 {
0x00004521 0xABCD 0xEF01   ──DECODE──▶     "table": "users",
0x88776655 0x4433 0x2211                   "action": "INSERT",
                                           "data": {"name": "Grok"}
Binary soup! 🤮                          }
                                         
                                         JSON! Me understand! 😊
```

---

## 🏔️ PART 4: SETTING UP CDC (Step by Step Caveman Guide)

### STEP 1: Tell Postgres "Write More Detail in WAL"

```sql
-- postgresql.conf (or RDS parameter group)

wal_level = logical    -- 🔥 IMPORTANT! Default is "replica"
                       -- "logical" = write enough detail for CDC

max_replication_slots = 4    -- How many CDC readers allowed

max_wal_senders = 4          -- How many can read WAL at same time
```

**In AWS RDS/Aurora:**
```
Parameter Group:
  rds.logical_replication = 1    ← This enable logical WAL
```

### STEP 2: Create "Replication Slot" (Reserved Seat for CDC)

Replication slot = **"Save my place in leaf pile!"**

```sql
-- Create a slot (reserved reading position)
SELECT pg_create_logical_replication_slot(
    'my_cdc_slot',      -- Name of slot
    'pgoutput'          -- Output plugin (how to format)
);

-- Other plugins:
-- 'wal2json'  → Output as JSON
-- 'pgoutput'  → Native Postgres (for Debezium)
-- 'test_decoding' → Simple text (for testing)
```

```
WAL PILE (leaves stacked):
                                    
    📜 Change #1                    
    📜 Change #2                    
    📜 Change #3  ◄─── Replication Slot says:
    📜 Change #4       "Me reading here! Don't throw away!"
    📜 Change #5                    
    📜 Change #6  ◄─── New changes keep coming
    
Without slot: Postgres throw away old leaves (WAL segments)
With slot: Postgres keep leaves until CDC reader finish!
```

### STEP 3: Create Publication (What Tables to Watch)

```sql
-- "Me want watch these tables!"

-- Watch specific tables:
CREATE PUBLICATION my_publication 
FOR TABLE users, orders, products;

-- Or watch EVERYTHING:
CREATE PUBLICATION all_tables_publication 
FOR ALL TABLES;
```

### STEP 4: Give Permission to CDC User

```sql
-- Create special user for CDC
CREATE USER cdc_reader WITH REPLICATION LOGIN PASSWORD 'strong_password';

-- Grant access to tables
GRANT SELECT ON ALL TABLES IN SCHEMA public TO cdc_reader;

-- Grant usage on schema
GRANT USAGE ON SCHEMA public TO cdc_reader;
```

---

## 🦖 PART 5: CDC TOOLS (Which Caveman Tool to Use?)

### Option A: **Debezium** (Most Popular) 🏆

```
┌─────────────┐     ┌───────────┐     ┌─────────┐     ┌──────────┐
│  Postgres   │────▶│  Debezium │────▶│  Kafka  │────▶│ Consumer │
│     WAL     │     │ Connector │     │  Topic  │     │  Apps    │
└─────────────┘     └───────────┘     └─────────┘     └──────────┘
```

**Debezium Config (Simple Version):**
```json
{
  "name": "postgres-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "my-aurora-cluster.xxx.rds.amazonaws.com",
    "database.port": "5432",
    "database.user": "cdc_reader",
    "database.password": "strong_password",
    "database.dbname": "mydb",
    "database.server.name": "myserver",
    "table.include.list": "public.users,public.orders",
    "plugin.name": "pgoutput",
    "slot.name": "debezium_slot",
    "publication.name": "my_publication"
  }
}
```

**What Debezium Send to Kafka:**
```json
{
  "before": null,                        // Old data (null for INSERT)
  "after": {                             // New data
    "id": 1,
    "name": "Grok",
    "email": "grok@cave.com"
  },
  "source": {
    "version": "2.1.0",
    "connector": "postgresql",
    "ts_ms": 1699012345000,              // When change happen
    "txId": 12345,                       // Transaction ID
    "lsn": 23456789,                     // WAL position
    "table": "users",
    "schema": "public"
  },
  "op": "c",                             // c=create, u=update, d=delete
  "ts_ms": 1699012345123
}
```

### Option B: **AWS DMS** (Database Migration Service)

```
┌─────────────┐     ┌─────────┐     ┌──────────────┐
│  Postgres   │────▶│  DMS    │────▶│ Target DB /  │
│   Aurora    │     │  Task   │     │ S3 / Kinesis │
└─────────────┘     └─────────┘     └──────────────┘
```

Good for:
- AWS to AWS stuff
- No Kafka needed
- Simple setup in console

### Option C: **pg_logical / Built-in Logical Replication**

```sql
-- On SOURCE database:
CREATE PUBLICATION my_pub FOR TABLE users;

-- On TARGET database:
CREATE SUBSCRIPTION my_sub 
  CONNECTION 'host=source dbname=mydb user=cdc_reader password=xxx'
  PUBLICATION my_pub;
```

Good for: Postgres-to-Postgres only. Simple. Native.

---

## 🌋 PART 6: IMPORTANT CAVEMAN WARNINGS!

### Warning 1: 🚨 REPLICATION SLOT CAN EAT ALL DISK!

```
If CDC reader STOP reading, but slot still exist:

    WAL files keep growing...
    
    Day 1:   WAL = 1 GB     😊
    Day 2:   WAL = 5 GB     🤔
    Day 3:   WAL = 20 GB    😰
    Day 7:   WAL = 200 GB   🔥 DISK FULL! DATABASE CRASH!
```

**Fix:**
```sql
-- Monitor slot lag:
SELECT slot_name, 
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS lag_size
FROM pg_replication_slots;

-- If slot abandoned, DROP IT:
SELECT pg_drop_replication_slot('abandoned_slot');
```

**Set max WAL size:**
```
# postgresql.conf
max_slot_wal_keep_size = 10GB    -- Don't keep more than this for any slot
```

### Warning 2: 🦴 NOT ALL CHANGES CAPTURED BY DEFAULT!

**Problem: TOAST data (big columns)**
```sql
-- BIG text column
UPDATE posts SET title = 'new title';   -- body column not in WAL!

-- body is TOASTed (stored separately)
-- Only changed columns in WAL!
```

**Fix:**
```sql
ALTER TABLE posts REPLICA IDENTITY FULL;

-- Now ALL columns included in WAL for this table
```

### Warning 3: 🗿 DDL CHANGES (Schema Changes) NOT IN WAL!

```sql
ALTER TABLE users ADD COLUMN age INT;    -- This NOT in logical WAL!
DROP TABLE old_stuff;                     -- This NOT in logical WAL!
```

CDC only capture **data changes** (INSERT, UPDATE, DELETE).
Schema changes need **separate handling** (DDL triggers, schema registry).

### Warning 4: 🦣 PRIMARY KEY NEEDED FOR UPDATE/DELETE!

```sql
-- Table with no primary key:
CREATE TABLE bad_table (
    name TEXT,
    value INT
);

UPDATE bad_table SET value = 10 WHERE name = 'foo';
-- CDC: "Which row updated? Me confused! 🤯"
```

**Fix:**
```sql
-- Option 1: Add primary key (BEST)
ALTER TABLE bad_table ADD PRIMARY KEY (id);

-- Option 2: Use REPLICA IDENTITY FULL
ALTER TABLE bad_table REPLICA IDENTITY FULL;
-- Now full row data in WAL (slower, bigger WAL)
```

---

## 🗻 PART 7: PICTURE OF FULL SYSTEM

```
                            THE FULL CDC CAVE SYSTEM
═══════════════════════════════════════════════════════════════════════════


  ┌─────────────────────────────────────────────────────────────────────┐
  │                         POSTGRES / AURORA                           │
  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │
  │  │   users     │  │   orders    │  │  products   │   ◄── Tables    │
  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                 │
  │         │                │                │                         │
  │         └────────────────┴────────────────┘                         │
  │                          │                                          │
  │                          ▼                                          │
  │                    ┌──────────┐                                     │
  │                    │   WAL    │  ◄── All changes written here      │
  │                    │ 📜📜📜📜  │      (wal_level = logical)         │
  │                    └────┬─────┘                                     │
  │                         │                                           │
  │            ┌────────────┴────────────┐                             │
  │            ▼                         ▼                              │
  │   ┌─────────────────┐      ┌─────────────────┐                     │
  │   │ REPLICATION     │      │  PUBLICATION    │                     │
  │   │ SLOT            │      │  (which tables) │                     │
  │   │ "debezium_slot" │      │  "my_pub"       │                     │
  │   └────────┬────────┘      └────────┬────────┘                     │
  │            └────────────┬───────────┘                              │
  └─────────────────────────┼──────────────────────────────────────────┘
                            │
                            │ TCP Connection (replication protocol)
                            ▼
             ┌──────────────────────────────┐
             │        DEBEZIUM              │
             │     (CDC Connector)          │
             │                              │
             │  • Connects as replication   │
             │    client                    │
             │  • Reads logical WAL         │
             │  • Converts to JSON/Avro     │
             │  • Sends to Kafka            │
             └──────────────┬───────────────┘
                            │
                            ▼
             ┌──────────────────────────────┐
             │          KAFKA               │
             │                              │
             │  Topics:                     │
             │  ├─ myserver.public.users    │
             │  ├─ myserver.public.orders   │
             │  └─ myserver.public.products │
             └──────────────┬───────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
  ┌───────────┐      ┌────────────┐      ┌────────────┐
  │  ELASTIC  │      │  ANOTHER   │      │  ANALYTICS │
  │  SEARCH   │      │  DATABASE  │      │  (Spark/   │
  │           │      │            │      │   Flink)   │
  │ For fast  │      │ Read       │      │            │
  │ searching │      │ replica    │      │ Real-time  │
  └───────────┘      └────────────┘      │ dashboards │
                                         └────────────┘

```

---

## 🦴 SUMMARY STONE TABLET

```
╔═══════════════════════════════════════════════════════════════╗
║                    CAVEMAN REMEMBER:                          ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  WAL = Write-Ahead Log = Safety leaf before carving stone    ║
║                                                               ║
║  CDC = Watch the WAL, tell others about changes              ║
║                                                               ║
║  SETUP:                                                       ║
║    1. wal_level = logical                                    ║
║    2. Create replication slot                                ║
║    3. Create publication                                     ║
║    4. Connect CDC tool (Debezium)                            ║
║                                                               ║
║  DANGERS:                                                     ║
║    • Slot lag = disk fill = bad                              ║
║    • Need primary keys                                       ║
║    • DDL not captured                                        ║
║    • TOAST columns tricky                                    ║
║                                                               ║
║  TOOLS:                                                       ║
║    • Debezium (best for Kafka)                               ║
║    • AWS DMS (best for AWS-to-AWS)                           ║
║    • Native logical replication (PG-to-PG only)              ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---
