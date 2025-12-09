# SQLite schema and where the code inserts/reads each table

This document lists all SQLite tables created by the gateway (extracted from
`src/RD_DataStore_Sqlite.c`) with their CREATE TABLE statements, column lists,
and the functions in the codebase that insert/persist and read/unpersist data
for each table. Use this as a reference to find where the runtime writes data
into the database.

Note: temporary runtime-only structures (for example `temp_association_table`)
are NOT persisted in SQLite and are not listed here.

---

## Table: `nodes`

CREATE statement (from `RD_DataStore_Sqlite.c`):

```sql
CREATE TABLE IF NOT EXISTS nodes (
  nodeid INTEGER PRIMARY KEY,
  wakeUp_interval  INTEGER,
  lastAwake  INTEGER,
  lastUpdate  INTEGER,
  security_flags  INTEGER,
  mode  INTEGER,
  state  INTEGER,
  manufacturerID  INTEGER,
  productType  INTEGER,
  productID  INTEGER,
  nodeType  INTEGER,
  nAggEndpoints  INTEGER,
  name  BLOB,
  dsk  BLOB,
  node_version_cap_and_zwave_sw INTEGER,
  probe_flags INTEGER,
  properties_flags INTEGER,
  cc_versions  BLOB,
  node_is_zws_probed INTEGER
);
```

Columns (summary): nodeid (PK), wakeUp_interval, lastAwake, lastUpdate,
security_flags, mode, state, manufacturerID, productType, productID, nodeType,
nAggEndpoints, name (BLOB), dsk (BLOB), node_version_cap_and_zwave_sw,
probe_flags, properties_flags, cc_versions (BLOB), node_is_zws_probed

Code that writes/inserts rows:
- `rd_data_store_nvm_write(rd_node_database_entry_t *n)` — binds values to a
  prepared `INSERT INTO nodes VALUES(?, ?,..., ?)` statement (see
  `node_insert_stmt`) and steps the statement. This function iterates the
  `endpoints` too (inserting into `endpoints`).

Code that reads rows:
- `rd_data_store_read(nodeid_t nodeID)` — executes prepared `SELECT * FROM
  nodes WHERE nodeid = ?` and reads the columns using `sqlite3_column_*`.

Code that deletes rows:
- `rd_data_store_nvm_free(rd_node_database_entry_t *n)` — executes
  `DELETE FROM nodes WHERE nodeid = ?`.

---

## Table: `endpoints`

CREATE statement:

```sql
CREATE TABLE IF NOT EXISTS endpoints (
  endpointid  INTEGER ,
  nodeid      INTEGER KEY,
  info        BLOB,
  aggr        BLOB,
  name        BLOB,
  location    BLOB,
  state       INTEGER,
  installer_iconID       INTEGER,
  user_iconID INTEGER
);
```

Columns (summary): endpointid, nodeid, info (BLOB), aggr (BLOB), name (BLOB),
location (BLOB), state, installer_iconID, user_iconID

Code that writes/inserts rows:
- Inside `rd_data_store_nvm_write(rd_node_database_entry_t *n)` the prepared
  statement `ep_insert_stmt` (INSERT INTO endpoints VALUES(?,?,?,?,?,?,?,?,?))
  is executed for each endpoint attached to the node.

Code that reads rows:
- `rd_data_store_read(nodeid_t nodeID)` — runs prepared `SELECT * FROM
  endpoints WHERE nodeid = ?` and loops over results, allocating and filling
  `rd_ep_database_entry_t` from the BLOB/column data.

Code that deletes rows:
- `rd_data_store_nvm_free()` executes `DELETE FROM endpoints WHERE nodeid = ?`.

---

## Table: `network`

CREATE statement:

```sql
CREATE TABLE IF NOT EXISTS network (
  homeid    INTEGER      NOT NULL UNIQUE,
  nodeid    INTEGER ,
  version_major        INTEGER ,
  version_minor        INTEGER
);
```

Columns (summary): homeid (unique), nodeid, version_major, version_minor

Code that writes/inserts rows:
- `clear_database()` inserts network info on a cleared DB using:
  `INSERT INTO network VALUES(?,?,?,?)` (bound homeID, MyNodeID and schema
  version).

Code that reads rows:
- `data_store_init()` and `rd_data_store_version_get()` prepare/execute
  `SELECT * FROM network` to validate or read schema version/homeid/nodeid.

Code that deletes rows:
- `rd_data_store_invalidate()` calls `DELETE FROM network;` as part of
  clearing the DB.

---

## Table: `ip_association`

CREATE statement:

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
```

Columns (summary): virtual_id, type, resource_ip (BLOB; usually 16-byte IPv6),
resource_endpoint, resource_port, virtual_endpoint, grouping, han_nodeid,
han_endpoint, was_dtls, mark_removal

Code that writes/inserts rows (persist):
- `rd_data_store_persist_associations(list_t ip_association_table)` — this
  function clears the table (`DELETE FROM ip_association;`) and then prepares
  the SQL `INSERT INTO ip_association VALUES(?,?,?,?,?,?,?,?,?,?,?)` and binds
  each `ip_association_t` element (including `sqlite3_bind_blob` for
  `resource_ip`).

Code that reads/unpersists rows:
- `rd_datastore_unpersist_association(list_t ip_association_table, struct memb *ip_association_pool)`
  — prepares `SELECT * FROM ip_association;`, steps rows, allocates
  `ip_association_t` objects and copies the BLOB into `a->resource_ip` if the
  blob size matches `sizeof(uip_ip6addr_t)`.

Notes: the runtime `temp_association_table` (virtual_id -> client ip:port) is
NOT this table — `ip_association` stores permanent associations that persist
across reboots.

---

## Table: `virtual_nodes`

CREATE statement:

```sql
CREATE TABLE IF NOT EXISTS virtual_nodes (
  virtual_id INTEGER PRIMARY KEY
);
```

Columns (summary): virtual_id (PK)

Code that writes/inserts rows (persist):
- `rd_datastore_persist_virtual_nodes(const nodeid_t *nodelist, size_t node_count)`
  — clears table via `DELETE FROM virtual_nodes` then prepares
  `INSERT INTO virtual_nodes VALUES(?)` and steps for each id in `nodelist`.

Code that reads/unpersists rows:
- `rd_datastore_unpersist_virtual_nodes(nodeid_t *nodelist, size_t max_node_count)`
  — prepares `SELECT * FROM virtual_nodes;` and reads `virtual_id` into the
  provided buffer.

---

## Table: `gw`

CREATE statement:

```sql
CREATE TABLE IF NOT EXISTS gw (
  mode INTEGER,
  showlock INTEGER,
  peerProfile INTEGER
);
```

Code that writes/inserts rows:
- `rd_datastore_persist_gw_config(const Gw_Config_St_t *gw_cfg)` — executes
  `DELETE FROM gw;` then `INSERT INTO gw VALUES(?,?,?)` bound with gw config
  fields.

Code that reads rows:
- `rd_datastore_unpersist_gw_config(Gw_Config_St_t *gw_cfg)` — `SELECT * FROM gw;`

---

## Table: `peer`

CREATE statement:

```sql
CREATE TABLE IF NOT EXISTS peer (
  id INTEGER PRIMARY KEY,
  ip BLOB,
  port INTEGER,
  name TEXT
);
```

Columns (summary): id (PK), ip (BLOB), port, name (TEXT)

Code that writes/inserts rows:
- `rd_datastore_persist_peer_profile(int index, const Gw_PeerProfile_St_t *profile)`
  — deletes existing row for id then prepares `INSERT INTO peer VALUES(?,?,?,?)`,
  binds `peer_col_ip` as BLOB (IPv6), `peer_col_port` (combined bytes) and
  `name`.

Code that reads rows:
- `rd_datastore_unpersist_peer_profile(int index, Gw_PeerProfile_St_t *profile)`
  — `SELECT * FROM peer WHERE id = ?;` then copies blob to the profile struct.

---

## Table: `s2_span`

CREATE statement:

```sql
CREATE TABLE IF NOT EXISTS s2_span (
  d BLOB,
  lnode INTEGER,
  rnode INTEGER,
  rx_seq INTEGER,
  tx_seq INTEGER,
  class_id INTEGER,
  state INTEGER
);
```

Columns (summary): d (BLOB), lnode, rnode, rx_seq, tx_seq, class_id, state

Code that writes/inserts rows:
- `rd_datastore_persist_s2_span_table(const struct SPAN *span_table, size_t span_table_size)`
  — clears the table (`DELETE FROM s2_span;`) and inserts rows with
  `INSERT INTO s2_span VALUES(?,?,?,?,?,?,?);` for entries where
  `entry->state == SPAN_NEGOTIATED`.

Code that reads rows:
- `rd_datastore_unpersist_s2_span_table(struct SPAN *span_table, size_t span_table_size)`
  — `SELECT * from s2_span;` and copies each row into the `SPAN` structures.

---

## Helper/utility usage in code
- `datastore_exec_sql(const char *sql)` — helper used to execute simple SQL
  strings (CREATE TABLE, DELETE FROM ...) and prints errors on failure.
- Prepared statements used in `prepare_statements()` create reusable INSERT/SELECT
  statements for `nodes`/`endpoints` paths.

---

## Quick reference: common INSERT strings (exact text from the code)
- `INSERT INTO network VALUES(?,?,?,?)` — in `clear_database()`
- `INSERT INTO nodes VALUES(?, ?,..., ?)` — prepared in `prepare_statements()`
- `INSERT INTO endpoints VALUES(?,?,?,?,?,?,?,?,?)` — prepared in `prepare_statements()`
- `INSERT INTO ip_association VALUES(?,?,?,?,?,?,?,?,?,?,?)` — in
  `rd_data_store_persist_associations`
- `INSERT INTO virtual_nodes VALUES(?)` — in
  `rd_datastore_persist_virtual_nodes`
- `INSERT INTO gw VALUES(?,?,?)` — in `rd_datastore_persist_gw_config`
- `INSERT INTO peer VALUES(?,?,?,?)` — in
  `rd_datastore_persist_peer_profile`
- `INSERT INTO s2_span VALUES(?,?,?,?,?,?,?);` — in
  `rd_datastore_persist_s2_span_table`

---

If you want, I can now:

- create `doc/test_virtual_nodes.db` containing example rows for
  `virtual_nodes`, `ip_association`, `nodes` to let you run the queries above,
  or
- add small code comments at the top of `src/RD_DataStore_Sqlite.c` mapping each
  function to the table(s) it uses (useful when reading the source), or
- prepare a short script (Python) that opens the DB and pretty-prints BLOB IP
  fields as IPv6 addresses.

Which would you like me to do next?
