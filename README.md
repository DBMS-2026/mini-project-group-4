# 🗄️ MiniDB Engine

![MiniDB Banner](main.png)

### A Relational Database Built From Scratch in C++

*Disk storage · B+ Tree indexing · LRU buffer pool · Custom query engine — zero dependencies on existing databases*

[![Language: C++](https://img.shields.io/badge/Language-C++17-blue?style=flat-square&logo=cplusplus)](https://isocpp.org/)
[![Build: Make](https://img.shields.io/badge/Build-Make%20%2F%20GCC-orange?style=flat-square)]()
[![Interface: Terminal](https://img.shields.io/badge/Interface-Terminal-black?style=flat-square)]()
[![Status: Active](https://img.shields.io/badge/Status-Active%20Development-green?style=flat-square)]()

---

## 🚀 Overview

MiniDB is a ground-up relational database engine. It does not wrap SQLite or MySQL; instead, it manages its own table metadata, page-based row storage, B+ Tree indexing, buffer pool caching, SQL-like parsing, and WAL recovery logging.

### Key Capabilities
- **Authenticated Access:** Secure CLI login and first-user bootstrap.
- **SQL Support:** `CREATE`, `INSERT`, `SELECT`, `UPDATE`, `DELETE`, `SHOW TABLES`, `DROP`.
- **Advanced Storage:** Slotted-page architecture with `RID(page_id, slot_id)` addressing.
- **High Performance:** Persistent B+ Tree index on primary keys for O(log N) lookups.
- **Memory Management:** Buffer-pool-backed read/write paths with LRU eviction.
- **Resiliency:** WAL-style REDO recovery on restart to ensure data integrity.
- **Integration:** Authenticated REST API (Crow-based) and Docker packaging.

---

## 🛠️ Getting Started

### Prerequisites
- GCC/G++ (C++17 support)
- Make
- Docker (optional)

### Build and Run
```bash
# Compile everything
make clean && make

# Start the interactive CLI
./miniDB -u <username> -p
```

### Build the API Server
```bash
make api
./server
```

---

## 📺 Feature Showcase

We have included a dedicated showcase script to demonstrate internal DBMS concepts:

```bash
# Run the Buffer Pool Visualization & Recovery Demo Instructions
./tests/run_aryan_feature_showcase.sh
```

### What's inside the showcase?
1.  **Buffer Pool Visualization:** Real-time table showing frame allocation, pin counts, dirty bits, and LRU eviction.
2.  **WAL Recovery Demo:** Step-by-step instructions to simulate a system crash and observe automatic REDO recovery upon restart.

---

## 🏗️ Architecture

```text
CLI / Query Console / API
   │
   ▼
Query Parser ──────▶ Execution Path
   │                     │
   ▼                     ▼
B+ Tree Index ◀───▶ Buffer Pool Mgr ◀───▶ Disk Manager
(index.dat)        (RAM Frames)          (data.dat)
```

### Storage Internals
- **Slotted Pages:** Efficiently packs rows into 4KB blocks.
- **Tuple Serialization:** Custom binary encoding for `INT` and `VARCHAR`.
- **WAL (Write-Ahead Log):** Ensures all modifications are logged before being applied to the data files.

---

## 🐳 Docker Deployment

MiniDB is fully containerized for portable deployment.

```bash
# Build and run using Docker Compose
docker compose up --build
```

The API server will be available at `http://localhost:18080`.

---

## 📜 Documentation
- [Final Project Report (FINAL.md)](./FINAL.md) - Deep dive into architecture and concepts.
- [REST API Reference](./Documentation/api.md) - Endpoint details and authentication.
- [Database Syntax](./syntax.md) - Supported SQL commands and types.

---

<div align="center">
<sub>Built to understand how a DBMS actually works internally.</sub>
</div>
