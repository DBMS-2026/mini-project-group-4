just # MiniDB Engine - Final Project Documentation

## Technical Overview & Project Architecture

MiniDB is a high-performance relational database engine built from the ground up in C++17. Unlike typical wrapper applications, MiniDB implements its own storage engine, memory management layers, and indexing structures, mirroring the internal design of professional systems like PostgreSQL or InnoDB.

---

### 1. Project Vision
MiniDB was designed to transition from a simple file-based parser to a robust, page-organized database system. The core focus was on implementing the fundamental layers of a DBMS:
*   **Physical Storage:** Fixed-size page architecture.
*   **Memory Management:** Buffer Pool with LRU replacement.
*   **Indexing:** Persistent B+ Tree for logarithmic lookup.
*   **Resiliency:** WAL-based REDO recovery for crash consistency.

---

### 2. Core Architectural Layers

#### A. Storage Engine (Physical Layer)
MiniDB uses a **Slotted-Page Storage Model**. 
*   **Page-Based I/O:** All data is managed in 4KB blocks (pages), which is the industry standard for efficient disk access.
*   **Record ID (RID):** Every row is uniquely identified by an `RID(page_id, slot_id)`.
*   **Slotted Layout:** Each page contains a header and a slot directory that grows forward, while tuple data grows backward from the end of the page, maximizing space utilization and handling variable-length records (VARCHAR).

#### B. Buffer Pool Manager (Memory Layer)
The **Buffer Pool** acts as an intelligent cache between the execution engine and the disk.
*   **LRU Replacement:** Implements the Least Recently Used policy to keep "hot" pages in RAM.
*   **Dirty Tracking:** Pages modified in memory are marked as "dirty" and only flushed to disk when necessary or during a checkpoint.
*   **Pinning Mechanism:** Ensures that pages currently being read or written are not evicted from memory, preventing data corruption.

#### C. B+ Tree Index (Access Layer)
MiniDB implements a **Persistent B+ Tree** on the primary key (the first column).
*   **O(log N) Performance:** Enables rapid point lookups without scanning the entire table.
*   **Persistence:** The tree structure is serialized and stored on disk, surviving system restarts.
*   **RID Mapping:** The leaf nodes map primary keys directly to their physical `RID`, linking the logical index to the physical storage.

#### D. WAL & Recovery (Resiliency Layer)
The **Write-Ahead Logging (WAL)** system ensures atomicity and durability.
*   **REDO Recovery:** Before any data page is updated, a log record is written. In the event of a crash, the system replays these logs on startup to restore the database to a consistent state.
*   **Crash Failpoints:** Includes deterministic failpoints for demonstration, proving that data survives even if the process is killed during a write.

---

### 3. Key Components & Implementation

| Component | Responsibility | Key File(s) |
|---|---|---|
| **Query Parser** | Tokenization and command routing | `parser.cpp` |
| **Buffer Manager** | Memory caching and page replacement | `buffer_pool_manager.cpp` |
| **Disk Manager** | Raw file I/O and page allocation | `disk_manager.cpp` |
| **Data Page** | Intrapage organization (slotted pages) | `data_page.cpp` |
| **B+ Tree** | Index management and lookup | `BPtree.cpp` |
| **Recovery Mgr** | WAL logging and REDO replay | `recovery_manager.cpp` |
| **Serializer** | Logic-to-physical byte conversion | `tuple_serializer.cpp` |

---

### 4. Advanced Features for Demo

#### 1. Buffer Pool Visualization
Using the `./tests/run_aryan_feature_showcase.sh` tool, you can visualize the internal state of the buffer pool in real-time. This shows frame allocation, pin counts, and the LRU eviction process as the system handles more data than can fit in RAM.

#### 2. Deterministic Crash Recovery
By setting the `MINIDB_CRASH_AFTER_WAL` environment variable, you can force the system to crash at the most critical moment. Upon restart, the system automatically detects the pending log records and applies them, demonstrating professional-grade recovery logic.

#### 3. Dynamic Tuple Relocation
When a row is updated with a larger value (e.g., a longer VARCHAR) that no longer fits in its current slot, MiniDB automatically relocates the tuple to a new page or slot, updates the B+ Tree index, and maintains full consistency across all layers.

---

### 5. Future Roadmap
*   **Concurrency Control:** Implementation of 2PL (Two-Phase Locking).
*   **Query Optimization:** Cost-based optimizer for complex queries.
*   **Join Engine:** Full support for hash-joins and nested-loop joins across multiple tables.

---
**MiniDB Engine** - *Built for understanding, designed for performance.*
