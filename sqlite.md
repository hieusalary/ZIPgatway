# SQLite layout for Virtual Nodes and IP Associations

This document describes the SQLite tables used by the gateway for virtual nodes and IP associations, the RAM temporary association structure, and example snapshots of the tables at different lifecycle stages.

Files referenced in repo:
- `src/RD_DataStore_Sqlite.c` — contains the DB schema (CREATE TABLE statements) and persisting/unpersisting functions.
- `src/Bridge_temp_assoc.c` — manages preallocated virtual node ids and temporary associations (RAM).
- `src/Bridge_ip_assoc.c` — creates and persists permanent IP associations (written to `ip_association` table).

---

## DB schema (as created in `RD_DataStore_Sqlite.c`)

CREATE TABLE statements (extracted from repo):

```sql
CREATE TABLE IF NOT EXISTS ip_association (
  virtual_id INTEGER,
  type INTEGER,
  resource_ip BLOB,
  resource_endpoint INTEGER,
  resource_port INTEGER,
  virtual_endpoint INTEGER,
  grouping INTEGER,
  han_nodeid INTEGER,
  han_endpoint INTEGER,
  was_dtls INTEGER,
  mark_removal INTEGER
);

CREATE TABLE IF NOT EXISTS virtual_nodes (
  virtual_id INTEGER PRIMARY KEY
);
```

Notes:
- `virtual_nodes` stores the list of preallocated classic virtual node IDs (one column, primary key).
- `ip_association` stores permanent IP associations including `virtual_id` and `resource_ip`/port/etc.
- Temporary associations (created on‑demand) are stored only in RAM (`temp_association_table`) — they are not written to the DB unless the client creates a permanent association.

---

## Column descriptions

virtual_nodes
- virtual_id INTEGER PRIMARY KEY
  - A single virtual node id per row. These IDs are typically preallocated on startup and persisted here so they are available after reboot.

ip_association
- virtual_id INTEGER
  - The virtual node id used by the controller for this association.
- type INTEGER
  - association type (e.g. PERMANENT_IP_ASSOC, PROXY_IP_ASSOC, LOCAL_IP_ASSOC). Mapped to constants in code.
- resource_ip BLOB
  - IPv6 address stored as blob (uip_ip6addr_t).
- resource_endpoint INTEGER
  - LAN endpoint for the resource.
- resource_port INTEGER
  - Port number (stored as integer; code uses host or network order as needed).
- virtual_endpoint INTEGER
  - Virtual node endpoint (if used).
- grouping INTEGER
  - grouping identifier (association group).
- han_nodeid INTEGER
  - HAN node id (the Z‑Wave node exposing association source).
- han_endpoint INTEGER
  - HAN endpoint.
- was_dtls INTEGER
  - Whether the resource connection should use DTLS (0/1).
- mark_removal INTEGER
  - Temporary flag used when removing associations (0/1).

---

## RAM temporary association struct (not persisted)

From `src/Bridge_temp_assoc.h` — `temp_association_t` fields:

- `virtual_id` (nodeid_t)
  - Active id used as source/destination when sending/receiving for this client (equals `virtual_id_static` normally, or `MyNodeID` for unsolicited dest).
- `virtual_id_static` (nodeid_t)
  - The preallocated virtual id assigned to this temp assoc (never changes while the struct exists).
- `resource_ip` (uip_ip6addr_t)
  - The client's IPv6 address.
- `resource_endpoint` (uint8_t)
  - The client's endpoint (ZIP payload endpoint).
- `resource_port` (uint16_t)
  - The client's source port (network byte order in struct comments).
- `was_dtls` (uint8_t)
  - Whether the incoming packet was DTLS.
- `is_long_range` (bool)
  - LR flag.

Temporary associations are stored in `temp_association_table` (a LIST) and manipulated by functions in `Bridge_temp_assoc.c`.

---

## Example snapshots (explicit rows & columns)

Below are 3 example snapshots (concrete values used for illustration).  Values are examples and may differ on an actual device.

### Snapshot A — Fresh DB (before preallocation)

virtual_nodes (DB)

| rowid | virtual_id |
| ----: | ---------: |
| (none) | (empty) |

ip_association (DB)

| virtual_id | type | resource_ip | resource_endpoint | resource_port | virtual_endpoint | grouping | han_nodeid | han_endpoint | was_dtls | mark_removal |
| ---: | ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| (none) | — | — | — | — | — | — | — | — | — | — |

temp_association_table (RAM)

| virtual_id | virtual_id_static | resource_ip | resource_port | resource_endpoint | was_dtls | is_long_range |
| ---: | ---: | --- | ---: | ---: | ---: | ---: |
| (none) | (none) | — | — | — | — | — |

---

### Snapshot B — After preallocation (bridge startup persisted IDs)

Suppose controller preallocates classic virtual ids 2,3,4,5 and `rd_datastore_persist_virtual_nodes()` wrote them to the DB.

virtual_nodes (DB)

| rowid | virtual_id |
| ----: | ---------: |
| 1 | 2 |
| 2 | 3 |
| 3 | 4 |
| 4 | 5 |

ip_association (DB)

| virtual_id | type | resource_ip | resource_endpoint | resource_port | virtual_endpoint | grouping | han_nodeid | han_endpoint | was_dtls | mark_removal |
| ---: | ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| (none persisted yet) | — | — | — | — | — | — | — | — | — | — |

temp_association_table (RAM)

| virtual_id | virtual_id_static | resource_ip | resource_port | resource_endpoint | was_dtls | is_long_range |
| ---: | ---: | --- | ---: | ---: | ---: | ---: |
| (none assigned to clients yet) | 2,3,4,5 exist in pool | — | — | — | — | — |

Notes:
- Preallocation writes only the virtual IDs to `virtual_nodes`. The pool is used later to assign `virtual_id_static` for temp associations.
- Long Range virtual ids (e.g. 4002..4005) may be managed in code (`lr_temp_assoc_virtual_nodeids[]`) and not necessarily persisted to `virtual_nodes`.

---

### Snapshot C — After a client (on‑demand temporary association)

Client A: IP `2001:db8::100`, port `40000` — first client uses `virtual_id_static = 2` from pool.

virtual_nodes (DB) — unchanged

| rowid | virtual_id |
| ----: | ---------: |
| 1 | 2 |
| 2 | 3 |
| 3 | 4 |
| 4 | 5 |

ip_association (DB) — unchanged (no permanent assoc)

| virtual_id | type | resource_ip | resource_endpoint | resource_port | ... |
| ---: | ---: | --- | ---: | ---: | --- |
| (none) | — | — | — | — | — |

temp_association_table (RAM)

| virtual_id | virtual_id_static | resource_ip | resource_port | resource_endpoint | was_dtls | is_long_range |
| ---: | ---: | --- | ---: | ---: | ---: | ---: |
| 2 | 2 | 2001:db8::100 | 40000 | 0 | 0 | false |

Notes:
- The mapping `virtual_id (2) -> resource_ip:2001:db8::100` is stored in RAM only. It is used by `CreateLogicalUDP()` to forward Z‑Wave replies to the client.
- `rd_datastore_persist_virtual_nodes()` may be called during this flow but it only writes the list of virtual IDs (not the IP mapping).

---

### Snapshot D — Permanent IP association persisted in `ip_association`

Client requests a permanent IP association; controller assigns virtual_id 3. The association is persisted to `ip_association`.

virtual_nodes (DB)
| rowid | virtual_id |
| ----: | ---------: |
| 1 | 2 |
| 2 | 3 |
| 3 | 4 |
| 4 | 5 |

ip_association (DB) — example persisted row

| virtual_id | type | resource_ip | resource_endpoint | resource_port | virtual_endpoint | grouping | han_nodeid | han_endpoint | was_dtls | mark_removal |
| ---: | ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 3 | 1 (PERMANENT_IP_ASSOC) | 2001:db8::200 | 1 | 5684 | 0 | 1 | 10 | 0 | 0 | 0 |

Meaning:
- Permanent mapping is persisted so the gateway can route replies to virtual_id=3 to `2001:db8::200:5684` across reboots.

---

## Quick reference: functions and locations
- Persist virtual node IDs: `temp_assoc_persist_virtual_nodeids()` calls `rd_datastore_persist_virtual_nodes()` (see `src/Bridge_temp_assoc.c`, `src/RD_DataStore_Sqlite.c`).
- Unpersist (load) virtual nodes: `temp_assoc_unpersist_virtual_nodeids()` calls `rd_datastore_unpersist_virtual_nodes()` on startup.
- Assign `virtual_id_static` to temp association: `temp_assoc_get_new_or_reuse_oldest()` reads from `temp_assoc_virtual_nodeids[]`.
- Create on‑demand mapping: `ClassicZIPUDP_input()` → `temp_assoc_create()` → `temp_assoc_setup_and_save()` (assigns `resource_ip`, `resource_port`, `virtual_id`).
- Use mapping for replies: `ZIP_Router` → `bridge_virtual_node_commandhandler()` → `CreateLogicalUDP()` → `temp_assoc_lookup_by_virtual_nodeid()` → `ZW_SendData_UDP()`.
- Persist permanent associations: `ip_assoc_setup_and_save()` → `ip_assoc_persist_association_table()` writes entries into `ip_association` table.

---

## Next actions (suggested)
- I can create a small example SQLite file (`test_virtual_nodes.db`) with the example rows above if you want to load and inspect it.
- Or I can add debug logging to `temp_assoc_setup_and_save()` to print mapping events at runtime (helpful for tracing mapping in a running gateway).

If you want the example DB file created in the repo, say so and I'll add it under `doc/` or `test/`.


# SQLite layout for Virtual Nodes and IP Associations

This document describes the SQLite tables used by the gateway for virtual nodes and IP associations, the RAM temporary association structure, and example snapshots of the tables at different lifecycle stages.

Files referenced in repo:
- `src/RD_DataStore_Sqlite.c` — contains the DB schema (CREATE TABLE statements) and persisting/unpersisting functions.
- `src/Bridge_temp_assoc.c` — manages preallocated virtual node ids and temporary associations (RAM).
- `src/Bridge_ip_assoc.c` — creates and persists permanent IP associations (written to `ip_association` table).

---

## DB schema (as created in `RD_DataStore_Sqlite.c`)

CREATE TABLE statements (extracted from repo):

```sql
CREATE TABLE IF NOT EXISTS ip_association (
  virtual_id INTEGER,
  type INTEGER,
  resource_ip BLOB,
  resource_endpoint INTEGER,
  resource_port INTEGER,
  virtual_endpoint INTEGER,
  grouping INTEGER,
  han_nodeid INTEGER,
  han_endpoint INTEGER,
  was_dtls INTEGER,
  mark_removal INTEGER
);

CREATE TABLE IF NOT EXISTS virtual_nodes (
  virtual_id INTEGER PRIMARY KEY
);
```

Notes:
- `virtual_nodes` stores the list of preallocated classic virtual node IDs (one column, primary key).
- `ip_association` stores permanent IP associations including `virtual_id` and `resource_ip`/port/etc.
- Temporary associations (created on‑demand) are stored only in RAM (`temp_association_table`) — they are not written to the DB unless the client creates a permanent association.

---

## Column descriptions

virtual_nodes
- virtual_id INTEGER PRIMARY KEY
  - A single virtual node id per row. These IDs are typically preallocated on startup and persisted here so they are available after reboot.

ip_association
- virtual_id INTEGER
  - The virtual node id used by the controller for this association.
- type INTEGER
  - association type (e.g. PERMANENT_IP_ASSOC, PROXY_IP_ASSOC, LOCAL_IP_ASSOC). Mapped to constants in code.
- resource_ip BLOB
  - IPv6 address stored as blob (uip_ip6addr_t).
- resource_endpoint INTEGER
  - LAN endpoint for the resource.
- resource_port INTEGER
  - Port number (stored as integer; code uses host or network order as needed).
- virtual_endpoint INTEGER
  - Virtual node endpoint (if used).
- grouping INTEGER
  - grouping identifier (association group).
- han_nodeid INTEGER
  - HAN node id (the Z‑Wave node exposing association source).
- han_endpoint INTEGER
  - HAN endpoint.
- was_dtls INTEGER
  - Whether the resource connection should use DTLS (0/1).
- mark_removal INTEGER
  - Temporary flag used when removing associations (0/1).

---

## RAM temporary association struct (not persisted)

From `src/Bridge_temp_assoc.h` — `temp_association_t` fields:

- `virtual_id` (nodeid_t)
  - Active id used as source/destination when sending/receiving for this client (equals `virtual_id_static` normally, or `MyNodeID` for unsolicited dest).
- `virtual_id_static` (nodeid_t)
  - The preallocated virtual id assigned to this temp assoc (never changes while the struct exists).
- `resource_ip` (uip_ip6addr_t)
  - The client's IPv6 address.
- `resource_endpoint` (uint8_t)
  - The client's endpoint (ZIP payload endpoint).
- `resource_port` (uint16_t)
  - The client's source port (network byte order in struct comments).
- `was_dtls` (uint8_t)
  - Whether the incoming packet was DTLS.
- `is_long_range` (bool)
  - LR flag.

Temporary associations are stored in `temp_association_table` (a LIST) and manipulated by functions in `Bridge_temp_assoc.c`.

---

## Example snapshots (explicit rows & columns)

Below are 3 example snapshots (concrete values used for illustration).  Values are examples and may differ on an actual device.

### Snapshot A — Fresh DB (before preallocation)

virtual_nodes (DB)

| rowid | virtual_id |
| ----: | ---------: |
| (none) | (empty) |

ip_association (DB)

| virtual_id | type | resource_ip | resource_endpoint | resource_port | virtual_endpoint | grouping | han_nodeid | han_endpoint | was_dtls | mark_removal |
| ---: | ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| (none) | — | — | — | — | — | — | — | — | — | — |

temp_association_table (RAM)

| virtual_id | virtual_id_static | resource_ip | resource_port | resource_endpoint | was_dtls | is_long_range |
| ---: | ---: | --- | ---: | ---: | ---: | ---: |
| (none) | (none) | — | — | — | — | — |

---

### Snapshot B — After preallocation (bridge startup persisted IDs)

Suppose controller preallocates classic virtual ids 2,3,4,5 and `rd_datastore_persist_virtual_nodes()` wrote them to the DB.

virtual_nodes (DB)

| rowid | virtual_id |
| ----: | ---------: |
| 1 | 2 |
| 2 | 3 |
| 3 | 4 |
| 4 | 5 |

ip_association (DB)

| virtual_id | type | resource_ip | resource_endpoint | resource_port | virtual_endpoint | grouping | han_nodeid | han_endpoint | was_dtls | mark_removal |
| ---: | ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| (none persisted yet) | — | — | — | — | — | — | — | — | — | — |

temp_association_table (RAM)

| virtual_id | virtual_id_static | resource_ip | resource_port | resource_endpoint | was_dtls | is_long_range |
| ---: | ---: | --- | ---: | ---: | ---: | ---: |
| (none assigned to clients yet) | 2,3,4,5 exist in pool | — | — | — | — | — |

Notes:
- Preallocation writes only the virtual IDs to `virtual_nodes`. The pool is used later to assign `virtual_id_static` for temp associations.
- Long Range virtual ids (e.g. 4002..4005) may be managed in code (`lr_temp_assoc_virtual_nodeids[]`) and not necessarily persisted to `virtual_nodes`.

---

### Snapshot C — After a client (on‑demand temporary association)

Client A: IP `2001:db8::100`, port `40000` — first client uses `virtual_id_static = 2` from pool.

virtual_nodes (DB) — unchanged

| rowid | virtual_id |
| ----: | ---------: |
| 1 | 2 |
| 2 | 3 |
| 3 | 4 |
| 4 | 5 |

ip_association (DB) — unchanged (no permanent assoc)

| virtual_id | type | resource_ip | resource_endpoint | resource_port | ... |
| ---: | ---: | --- | ---: | ---: | --- |
| (none) | — | — | — | — | — |

temp_association_table (RAM)

| virtual_id | virtual_id_static | resource_ip | resource_port | resource_endpoint | was_dtls | is_long_range |
| ---: | ---: | --- | ---: | ---: | ---: | ---: |
| 2 | 2 | 2001:db8::100 | 40000 | 0 | 0 | false |

Notes:
- The mapping `virtual_id (2) -> resource_ip:2001:db8::100` is stored in RAM only. It is used by `CreateLogicalUDP()` to forward Z‑Wave replies to the client.
- `rd_datastore_persist_virtual_nodes()` may be called during this flow but it only writes the list of virtual IDs (not the IP mapping).

---

### Snapshot D — Permanent IP association persisted in `ip_association`

Client requests a permanent IP association; controller assigns virtual_id 3. The association is persisted to `ip_association`.

virtual_nodes (DB)
| rowid | virtual_id |
| ----: | ---------: |
| 1 | 2 |
| 2 | 3 |
| 3 | 4 |
| 4 | 5 |

ip_association (DB) — example persisted row

| virtual_id | type | resource_ip | resource_endpoint | resource_port | virtual_endpoint | grouping | han_nodeid | han_endpoint | was_dtls | mark_removal |
| ---: | ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 3 | 1 (PERMANENT_IP_ASSOC) | 2001:db8::200 | 1 | 5684 | 0 | 1 | 10 | 0 | 0 | 0 |

Meaning:
- Permanent mapping is persisted so the gateway can route replies to virtual_id=3 to `2001:db8::200:5684` across reboots.

---

## Quick reference: functions and locations
- Persist virtual node IDs: `temp_assoc_persist_virtual_nodeids()` calls `rd_datastore_persist_virtual_nodes()` (see `src/Bridge_temp_assoc.c`, `src/RD_DataStore_Sqlite.c`).
- Unpersist (load) virtual nodes: `temp_assoc_unpersist_virtual_nodeids()` calls `rd_datastore_unpersist_virtual_nodes()` on startup.
- Assign `virtual_id_static` to temp association: `temp_assoc_get_new_or_reuse_oldest()` reads from `temp_assoc_virtual_nodeids[]`.
- Create on‑demand mapping: `ClassicZIPUDP_input()` → `temp_assoc_create()` → `temp_assoc_setup_and_save()` (assigns `resource_ip`, `resource_port`, `virtual_id`).
- Use mapping for replies: `ZIP_Router` → `bridge_virtual_node_commandhandler()` → `CreateLogicalUDP()` → `temp_assoc_lookup_by_virtual_nodeid()` → `ZW_SendData_UDP()`.
- Persist permanent associations: `ip_assoc_setup_and_save()` → `ip_assoc_persist_association_table()` writes entries into `ip_association` table.

---

## Next actions (suggested)
- I can create a small example SQLite file (`test_virtual_nodes.db`) with the example rows above if you want to load and inspect it.
- Or I can add debug logging to `temp_assoc_setup_and_save()` to print mapping events at runtime (helpful for tracing mapping in a running gateway).

If you want the example DB file created in the repo, say so and I'll add it under `doc/` or `test/`.

---

## Visual diagrams (quick reference)

Below are two diagrams to help visualize how virtual node IDs and associations flow between the DB, RAM and packet processing paths.

1) High‑level storage & mapping (DB vs RAM)

```
  +----------------+        persist/load        +----------------+
  | virtual_nodes  | <------------------------> | temp_assoc_    |
  |  (SQLite DB)   |   rd_datastore_* funcs    | virtual_nodeids |
  |----------------|                           | (RAM array)     |
  | virtual_id PK  |                           +----------------+
  +----------------+                                    |
                v
            +-------------------------------+
            | temp_association_table (RAM)  |
            |-------------------------------|
            | virtual_id  | resource_ip     |
            | virtual_id_ | resource_port   |
            | static      | endpoint        |
            +-------------------------------+

Notes:
- `virtual_nodes` in DB stores the preallocated IDs (one column). On startup they are loaded into the RAM array `temp_assoc_virtual_nodeids[]`.
- `temp_association_table` holds the actual mapping virtual_id -> resource_ip:port used at runtime (not persisted unless made permanent).

2) Packet flow (IP client <-> gateway <-> Z‑Wave node)

```
  [Z/IP client IP:port]                       [Z-Wave node]
    |                                        ^
    | UDP ZIP (client -> GW)                 |
    v                                        |
  +----------------------+                       |
  | ClassicZIPUDP_input  |                       |
  | (ClassicZIPNode.c)   |                       |
  +----------------------+                       |
    |                                        |
    | temp_assoc_create() -> temp_assoc_setup_and_save()
    | (assign virtual_id_static from pool)   |
    v                                        |
  +-------------------------------+              |
  | temp_association_table (RAM)  |              |
  | (virtual_id -> resource_ip)   |              |
  +-------------------------------+              |
    |                                        |
    | send_using_temp_assoc()                |
    | (ZW_SendDataAppl with p.snode = virtual_id)
    v                                        |
  [sent on Z-Wave with source = virtual_id] ---->+

  Reply from Z-Wave node (dst = virtual_id) flows back into the gateway:

  [Z-Wave node] --> (serial) --> ZIP_Router.c receives frame
    |
    v
  ZIP_Router: (p->dnode != MyNodeID) => bridge_virtual_node_commandhandler()
    |
    v
  bridge_virtual_node_commandhandler() -> CreateLogicalUDP()
    |
    v
  CreateLogicalUDP() does: a = temp_assoc_lookup_by_virtual_nodeid(p->dnode)
  if (a) -> ZW_SendData_UDP(&c) to a->resource_ip:a->resource_port

```

Mermaid alternative (if your viewer supports it):

```mermaid
flowchart LR
  subgraph DB
    VN[(virtual_nodes\nSQLite)]
    IPA[(ip_association\nSQLite)]
  end
  VN -->|unpersist| Pool[temp_assoc_virtual_nodeids[] (RAM)]
  Pool -->|assign| TempRAM[temp_association_table (RAM)\nvirtual_id -> resource_ip]
  TempRAM -->|used by| ClassicZIP[ClassicZIPUDP_input]
  ClassicZIP -->|ZW_SendDataAppl(src=virtual_id)| ZWave[Z-Wave network]
  ZWave -->|reply dst=virtual_id| ZIPRouter[ZIP_Router]
  ZIPRouter -->|bridge handler| CreateLogicalUDP[CreateLogicalUDP]
  CreateLogicalUDP -->|lookup| TempRAM
  CreateLogicalUDP -->|ZW_SendData_UDP| UDPClient[Z/IP client]

  click VN href "../src/RD_DataStore_Sqlite.c" "Open RD_DataStore_Sqlite.c"
  click Pool href "../src/Bridge_temp_assoc.c" "Open Bridge_temp_assoc.c"
  click TempRAM href "../src/Bridge_temp_assoc.h" "Open Bridge_temp_assoc.h"
```

---

If you want, I can also create the example SQLite file (`test_virtual_nodes.db`) in `doc/` matching the snapshots above so you can inspect it with the `sqlite3` CLI or a GUI viewer.

