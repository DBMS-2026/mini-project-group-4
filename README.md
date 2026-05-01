# MiniDB Engine

MiniDB is a relational database engine built from scratch in C++. It does not wrap SQLite or MySQL. It manages its own table metadata, page-based row storage, B+ Tree indexing, buffer pool caching, SQL-like parsing, recovery logging, and a small authenticated REST API.

This repository is currently at the stage where the core engine is working end to end:

- authenticated CLI login and first-user bootstrap
- `CREATE`, `INSERT`, `SELECT`, `SHOW TABLES`, `UPDATE`, `DROP`
- page-based storage with slotted pages and `RID(page_id, slot_id)`
- persistent B+ Tree index on the primary key
- buffer-pool-backed read/write paths
- WAL-style REDO recovery on restart
- REST API server
- Docker packaging for portable local deployment

---

## Table of Contents

- [Project Status](#project-status)
- [Current Capabilities](#current-capabilities)
- [How It Works](#how-it-works)
- [Build and Run](#build-and-run)
- [Query Console](#query-console)
- [Search Behavior](#search-behavior)
- [Storage Architecture](#storage-architecture)
- [Recovery Layer](#recovery-layer)
- [REST API](#rest-api)
- [Docker](#docker)
- [Repository Layout](#repository-layout)
- [Tech Stack](#tech-stack)
- [Known Scope Limits](#known-scope-limits)

---

## Project Status

MiniDB is not a toy parser sitting on text files anymore. The current `main` branch has a functioning storage engine core with:

- fixed-size page storage in `data.dat`
- row addressing through `RID`
- buffer pool caching with dirty tracking and checkpoints
- B+ Tree primary-key lookup
- update support, including tuple relocation when the new row no longer fits in the old slot
- startup recovery through WAL replay

It is still a learning/project DBMS, not a production-ready database. That distinction matters. The storage and execution model is real, but advanced production concerns like concurrency control, transaction isolation, and full ARIES-style recovery are not complete.

---

## Current Capabilities

### SQL-like Operations

The current Query Console supports:

- `SHOW TABLES;`
- `CREATE TABLE ...;`
- `INSERT INTO ... VALUES (...);`
- `SELECT * FROM ...;`
- `SELECT col1, col2 FROM ...;`
- `SELECT ... WHERE ...;`
- `UPDATE ... SET ... WHERE ...;`
- `DROP TABLE ...;`

### Interactive Menu

The terminal menu currently exposes:

1. Query Console
2. Search table / search inside table
3. Print metadata of a table
4. Help
5. Quit

### Storage and Execution Features

- Page-based row storage inside `table/<name>/data.dat`
- Table metadata stored separately in `table/<name>/met`
- Persistent B+ Tree stored in `table/<name>/index.dat`
- Slotted-page layout through `DataPage`
- Tuple serialization for `INT` and `VARCHAR`
- `RID(page_id, slot_id)` addressing
- Buffer pool manager with:
  - page table
  - pin count
  - dirty bit
  - LRU-style replacement
  - stats and checkpoint support
- WAL-style page REDO logging and startup recovery

### Authentication

MiniDB now has a system-level `auth` table.

- On the first run, it creates the initial user.
- On later runs, it authenticates the username and password.
- The REST API uses session tokens derived from login.

---

## How It Works

At a high level, the engine flow is:

```text
CLI / Query Console / API
-> Parser
-> Execution path
-> B+ Tree lookup or linear scan
-> Buffer Pool Manager
-> Disk Manager
-> data.dat / index.dat / met
```

For an insert:

```text
row values
-> TupleSerializer
-> DataPage insert into a slotted page
-> RID(page_id, slot_id)
-> B+ Tree key -> RID mapping
-> WAL record
-> flush through BufferPoolManager / DiskManager
```

For a primary-key select:

```text
WHERE id = ...
-> B+ Tree search
-> RID
-> BufferPoolManager fetch
-> DataPage read
-> TupleSerializer deserialize
-> print result
```

For a non-primary-key search:

```text
WHERE non_pk_col = ...
-> linear scan across pages
-> deserialize row by row
-> match filter
-> print result
```

---

## Build and Run

### Build the CLI

```bash
make
```

### Run the CLI

```bash
./miniDB -u <username> -p
```

Important behavior:

- if no users exist yet, MiniDB will create the initial user
- otherwise it will prompt for the password of the given username

Example:

```bash
./miniDB -u aryan -p
```

### Build the API server

```bash
make api
```

This builds:

- `miniDB`
- `server`

Then run:

```bash
./server
```

The API server listens on:

```text
http://localhost:18080
```

---

## Query Console

Once logged in, choose option `1` to enter the Query Console.

Supported examples:

```sql
SHOW TABLES;

CREATE TABLE students (id INT, name VARCHAR(50), dept VARCHAR(20));

INSERT INTO students VALUES (1, "Aditya", "CSE");
INSERT INTO students VALUES (2, "Aryan", "DBMS");

SELECT * FROM students;
SELECT name, dept FROM students;
SELECT * FROM students WHERE id = 1;

UPDATE students SET dept = ECE WHERE id = 1;

DROP TABLE students;
```

Notes:

- SQL keywords are case-insensitive
- table names and column names are case-sensitive
- `VARCHAR` values preserve original case
- first column must be `INT`
- the first column acts as the primary key index target

---

## Search Behavior

MiniDB uses two different search paths depending on the `WHERE` column:

### Primary-key search

If the `WHERE` column is the first `INT` column:

- the query routes to the B+ Tree
- the tree returns `RID(page_id, slot_id)`
- the engine fetches the page and reads the row directly

The CLI prints:

```text
[Search Strategy: B+ Tree Point Lookup on Primary Key]
```

### Non-primary-key search

If the `WHERE` column is not the primary key:

- the engine scans all pages linearly
- each tuple is deserialized
- rows are matched one by one

The CLI prints:

```text
[Search Strategy: Linear Scan]
```

---

## Storage Architecture

### 1. Table Metadata

Each table stores schema metadata in:

```text
table/<table_name>/met
```

This contains:

- table name
- column names
- column types
- column sizes
- record size
- record count

### 2. Data Pages

Actual row data is stored in:

```text
table/<table_name>/data.dat
```

Rows are not stored one-file-per-row. They are packed into fixed-size pages.

MiniDB currently uses:

- fixed-size page I/O
- slotted-page layout
- `PageHeader`
- `SlotEntry`
- `RID(page_id, slot_id)`

### 3. Tuple Serialization

Rows are converted to raw bytes before storage.

- `INT` -> fixed 4 bytes
- `VARCHAR` -> length-prefixed bytes

That conversion is handled by:

- [`tuple_serializer.cpp`](./src/tuple_serializer.cpp)

### 4. Buffer Pool

The buffer pool sits between the execution layer and disk I/O.

It manages:

- cached page frames
- `page_id -> frame_id` mapping
- dirty pages
- pin counts
- checkpointing
- replacement policy support

This means inserts, display, search, and current local update work on page copies in RAM first, then flush changes back to disk.

### 5. Index Storage

Primary-key indexing is persisted in:

```text
table/<table_name>/index.dat
```

The B+ Tree stores:

```text
primary_key -> RID(page_id, slot_id)
```

This gives fast point lookup for primary-key queries.

---

## Recovery Layer

MiniDB now has a simple WAL-style REDO recovery layer.

### What it does

Before certain page changes are finalized, the engine writes a recovery record that stores the page after-image needed to replay the operation later.

On restart:

- `recover_all_tables()` scans WAL files
- pending records are replayed
- data pages and index state are repaired

### What this currently covers

- recovery-aware insert path
- recovery-aware update path
- restart-time REDO replay

### What it is not yet

This is not a full transaction system. It is not full ARIES. It is not full undo/redo with concurrency.

The current layer is best described as:

- WAL-inspired
- REDO-based
- crash-demo-friendly

### Deterministic recovery testing

The engine also includes a crash failpoint for controlled demos:

```bash
MINIDB_CRASH_AFTER_WAL=insert ./miniDB -u <username> -p
MINIDB_CRASH_AFTER_WAL=update ./miniDB -u <username> -p
```

This intentionally crashes right after WAL logging so restart recovery can be demonstrated reliably.

---

## REST API

MiniDB exposes a Crow-based REST API through `api/server.cpp`.

### Build

```bash
make api
./server
```

### Base URL

```text
http://localhost:18080
```

### Available Routes

#### Authentication

- `POST /login`
- `POST /logout`

#### Table data

- `GET /tables`
- `GET /table/<table_name>`
- `GET /meta/<table_name>`
- `POST /create`
- `POST /insert/<table_name>`
- `POST /bulk_insert/<table_name>`

#### Health

- `GET /health`

### Authentication model

Most routes require:

```text
X-Session-Token
```

returned by `/login`.

### Example

```bash
curl -X POST http://localhost:18080/login \
  -H "Content-Type: application/json" \
  -d '{"username":"aryan","password":"your_password"}'
```

Then use the returned token:

```bash
curl http://localhost:18080/tables \
  -H "X-Session-Token: <token>"
```

For endpoint details, see [Documentation/api.md](./Documentation/api.md).

---

## Docker

MiniDB now includes Docker packaging so the current engine can be built and run in a reproducible environment.

### Files added

- [`Dockerfile`](./Dockerfile)
- [`docker-entrypoint.sh`](./docker-entrypoint.sh)
- [`.dockerignore`](./.dockerignore)
- [`docker-compose.yml`](./docker-compose.yml)

### What the image contains

The image builds and packages:

- `miniDB` -> interactive CLI
- `server` -> REST API server

### Build the image

```bash
docker build -t minidb-engine .
```

### Run the API container

```bash
docker run --rm -p 18080:18080 \
  -v "$(pwd)/docker-data/table:/app/table" \
  -v "$(pwd)/docker-data/system:/app/system" \
  minidb-engine
```

Health check:

```bash
curl http://localhost:18080/health
```

### Run with Docker Compose

```bash
docker compose up --build
```

### Run the CLI from the same image

```bash
docker run --rm -it \
  -v "$(pwd)/docker-data/table:/app/table" \
  -v "$(pwd)/docker-data/system:/app/system" \
  --entrypoint /app/miniDB \
  minidb-engine -u <username> -p
```

### Why the mounted volumes matter

MiniDB stores persistent state inside:

- `/app/table`
- `/app/system`

Those hold:

- table metadata
- row pages
- B+ Tree pages
- WAL/recovery files
- auth/system files

If you do not mount them, your state disappears when the container is removed.

### Scope note

This Docker layer is for:

- reproducible builds
- local testing
- easier evaluation/demo
- portable project packaging

It should not be described as proof of production scalability on its own.

---

## Repository Layout

```text
DBMS/
├── api/
│   └── server.cpp
├── include/
│   ├── auth.h
│   ├── buffer_pool_manager.h
│   ├── BPtree.h
│   ├── data_page.h
│   ├── disk_manager.h
│   ├── recovery_manager.h
│   ├── storage_types.h
│   ├── tuple_serializer.h
│   └── ...
├── src/
│   ├── auth.cpp
│   ├── buffer_pool_manager.cpp
│   ├── BPtree.cpp
│   ├── create.cpp
│   ├── data_page.cpp
│   ├── disk_manager.cpp
│   ├── display.cpp
│   ├── file_handler.cpp
│   ├── insert.cpp
│   ├── main.cpp
│   ├── parser.cpp
│   ├── recovery_manager.cpp
│   ├── tuple_serializer.cpp
│   ├── update.cpp
│   └── where.cpp
├── table/
├── system/
├── Dockerfile
├── docker-compose.yml
├── docker-entrypoint.sh
├── Makefile
└── README.md
```

---

## Tech Stack

| Area | Technology |
|---|---|
| Core engine | C++17 |
| Build system | Make + GCC/G++ |
| API layer | Crow |
| Storage format | custom page-based binary files |
| Indexing | custom persistent B+ Tree |
| Caching | custom buffer pool |
| Recovery | WAL-style REDO logging |
| Packaging | Docker + Docker Compose |

---

## Known Scope Limits

MiniDB is already substantial, but some DBMS-level work is still out of scope or incomplete:

- no full transaction manager
- no concurrency control / locking
- no isolation levels
- no full ARIES implementation
- no production-grade query optimizer
- no full SQL grammar
- no multi-table join support on current `main`

That said, the current repository is strong as a systems/database project because the core storage and execution layers are real and inspectable.

---

Built to understand how a DBMS actually works internally: parsing, storage, indexing, caching, recovery, and execution.
