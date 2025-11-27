# Node Data Storage in Z/IP Gateway

Tài liệu này liệt kê toàn bộ thông tin liên quan tới một Z-Wave node mà Z/IP Gateway lưu trữ: cấu trúc dữ liệu trong RAM, nơi persist (SQLite / NVM / file cấu hình), đoạn mã thực hiện việc ghi / cập nhật, và giải thích các dòng quan trọng. Bao gồm cả ví dụ NAT table và `rd_node_database_entry_t`.

## 1. Tổng quan phân lớp lưu trữ

| Nhóm dữ liệu | Thành phần | Nguồn gốc / Cập nhật | Persist ở đâu | Mục đích sử dụng chính |
|--------------|------------|----------------------|---------------|-------------------------|
| Thông tin node cơ bản | `nodeid`, `nodeType`, `mode`, `state` | NIF / Protocol Info / Probe state machine | SQLite bảng `nodes` (cột `nodeid`, `mode`, `state`) | Phân loại thiết bị, quyết định timeout & logic probe |
| Trạng thái probe | `wakeUp_interval`, `lastAwake`, `lastUpdate`, `probe_flags` | WakeUp CC, thời điểm nhận frame, tiến trình interview | SQLite `nodes` | Mailbox logic, đánh giá failing / scheduling |
| Security | `security_flags` (S0/S2 bits, KNOWN_BAD), SPAN table | KEX/S2 inclusion / downgrade trong probe NIF | `nodes.security_flags`; SPAN: bảng `s2_span` | Chọn scheme gửi, kiểm tra unsolicited, S2 duplication detection |
| DSK (SmartStart/S2) | `dsk`, `dskLen` | S2 xác thực / provisioning | `nodes.dsk` | Map với provisioning list, định danh thiết bị |
| Manufacturer/Product | `manufacturerID`, `productType`, `productID` | Manufacturer Specific CC Report | `nodes.manufacturerID/productType/productID` | Hiển thị UI, phân nhóm thiết bị |
| Command Class Versions | `node_cc_versions` + `node_cc_versions_len` | Version CC probing (`rd_ep_probe_cc_version`) | `nodes.cc_versions` | Quyết định version khi điều khiển CC |
| Z-Wave Plus Icons | `installer_iconID`, `user_iconID` (endpoint) | Z-Wave Plus Info Report | `endpoints.installer_iconID/user_iconID` | UI / phân loại role, portable flag |
| Endpoint cấu trúc | `endpoint_id`, `endpoint_info`, `endpoint_agg` | MULTI_CHANNEL_CAPABILITY_REPORT / AGGREGATED_MEMBERS | `endpoints.info`, `endpoints.aggr` | Kiểm tra hỗ trợ CC, encaps Multi Channel |
| Endpoint metadata | `endpoint_name`, `endpoint_location` | mDNS probe / Z/IP Naming CC SET/GET | `endpoints.name`, `endpoints.location` | Tên thân thiện, discovery LAN |
| Node name | `nodename`, `nodeNameLen` | mDNS probe / auto generate | `nodes.name` | Service discovery, hiển thị |
| NAT IPv4 mapping | `nat_table[nodeid → ip_suffix]` | `ipv46nat_add_entry()` / DHCP cấp IP | RAM (mảng `nat_table[]`) | Ánh xạ IPv4–Z-Wave node cho host IPv4 |
| Virtual nodes | Danh sách node ảo | Quản lý bridge/association | Bảng `virtual_nodes` | Forwarding, ánh xạ dịch vụ |
| IP Associations | `ip_association_t` entries | IP Association CC / quản lý cấu hình | Bảng `ip_association` | Dịch Association sang tài nguyên IP |
| SPAN (S2 session) | `struct SPAN { d, lnode, rnode, rx_seq, tx_seq, class_id, state }` | S2 negotiation/runtime | Bảng `s2_span` | Khôi phục trạng thái S2 sau restart |
| Keystore keys | Network keys per security class | Tạo mới / nhập khi inclusion | NVM (keystore), via `keystore_network_key_write()` | Mã hoá/giải mã S0/S2 |

## 2. Cấu trúc dữ liệu chính (RAM)

### 2.1 `rd_node_database_entry_t` (trích yếu phần quan trọng – xem đầy đủ trong `RD_internal.h`)
```c
typedef struct rd_node_database_entry {
  uint32_t wakeUp_interval;     // Khoảng wakeup đã biết
  uint32_t lastAwake;           // Lần cuối node thức
  uint32_t lastUpdate;          // Lần cuối probe thành công
  uint8_t  not_used[16];        // Vùng legacy
  nodeid_t nodeid;              // Node ID
  uint8_t  security_flags;      // Bitmask S0/S2/KNOWN_BAD ...
  rd_node_mode_t mode;          // Mode + flags (listening, mailbox...)
  rd_node_state_t state;        // Trạng thái probe FSM
  uint16_t manufacturerID;
  uint16_t productType;
  uint16_t productID;
  uint8_t  nodeType;            // Basic device type
  uint8_t  refCnt;
  uint8_t  nEndpoints;          // Tổng endpoint
  uint8_t  nAggEndpoints;       // Số aggregated EP
  LIST_STRUCT(endpoints);       // Danh sách rd_ep_database_entry_t
  uint8_t  nodeNameLen;
  uint8_t  dskLen;
  uint8_t  node_version_cap_and_zwave_sw; // Version capability + ZW SW report
  uint16_t probe_flags;         // Tổng hợp trạng thái probe
  uint16_t node_properties_flags; // Portable, Added_By_Me...
  uint8_t  node_cc_versions_len;
  uint8_t  node_is_zws_probed;
  char*    nodename;            // mDNS tên node
  uint8_t* dsk;                 // DSK nếu có
  cc_version_pair_t *node_cc_versions; // Cache versions các CC
  ProbeCCVersionState_t *pcvs;  // State nội bộ version probe (không persist)
} rd_node_database_entry_t;
```

### 2.2 `rd_ep_database_entry_t`
```c
typedef struct rd_ep_database_entry {
  struct rd_ep_database_entry* list; // quản lý list Contiki
  rd_node_database_entry_t* node;    // con trỏ về node cha
  char *endpoint_location;           // mDNS location
  char *endpoint_name;               // mDNS name / Z/IP Naming
  uint8_t *endpoint_info;            // NIF endpoint (CC list)
  uint8_t *endpoint_agg;             // Aggregated members
  uint8_t endpoint_info_len;
  uint8_t endpoint_name_len;
  uint8_t endpoint_loc_len;
  uint8_t endpoint_aggr_len;
  uint8_t endpoint_id;
  rd_ep_state_t state;
  uint16_t installer_iconID;
  uint16_t user_iconID;
} rd_ep_database_entry_t;
```

### 2.3 NAT Table (`ipv46_nat.c`)
```c
nat_table_entry_t nat_table[MAX_NAT_ENTRIES];
uint16_t nat_table_size;
// Mỗi entry: nodeid + ip_suffix (phần host IPv4)
```
Không persist: tái tạo khi cấp phát IPv4 / khởi động.

### 2.4 SPAN Table (S2 session) – nằm trong `s2_ctx->span_table`
Persist điều kiện state = `SPAN_NEGOTIATED`.

## 3. Code ghi dữ liệu node & endpoints (SQLite)

### 3.1 Ghi node + endpoints: `rd_data_store_nvm_write()` (`RD_DataStore_Sqlite.c`)
Đoạn gốc (rút gọn, giữ dòng quan trọng – đã bỏ phần lặp không cần thiết):
```c
void rd_data_store_nvm_write(rd_node_database_entry_t *n) {
  rd_ep_database_entry_t *e;
  // Xóa bản ghi cũ (node + endpoints)
  rd_data_store_nvm_free(n);

  sqlite3_reset(node_insert_stmt);
  sqlite3_bind_ex_int(node_insert_stmt, nt_col_nodeid, n->nodeid);
  sqlite3_bind_ex_int(node_insert_stmt, nt_col_wakeUp_interval, n->wakeUp_interval);
  sqlite3_bind_ex_int(node_insert_stmt, nt_col_lastAwake, n->lastAwake);
  sqlite3_bind_ex_int(node_insert_stmt, nt_col_lastUpdate, n->lastUpdate);
  sqlite3_bind_ex_int(node_insert_stmt, nt_col_security_flags, n->security_flags);       // Security flags
  sqlite3_bind_ex_int(node_insert_stmt, nt_col_mode, n->mode);                          // Mode + flags
  sqlite3_bind_ex_int(node_insert_stmt, nt_col_state, n->state);                        // FSM state
  sqlite3_bind_ex_int(node_insert_stmt, nt_col_manufacturerID, n->manufacturerID);
  sqlite3_bind_ex_int(node_insert_stmt, nt_col_productType, n->productType);
  sqlite3_bind_ex_int(node_insert_stmt, nt_col_productID, n->productID);
  sqlite3_bind_ex_int(node_insert_stmt, nt_col_nodeType, n->nodeType);
  sqlite3_bind_ex_int(node_insert_stmt, nt_col_nAggEndpoints, n->nAggEndpoints);
  sqlite3_bind_ex_int(node_insert_stmt, nt_col_node_version_cap_and_zwave_sw, n->node_version_cap_and_zwave_sw);
  sqlite3_bind_ex_int(node_insert_stmt, nt_col_probe_flags, n->probe_flags);
  sqlite3_bind_ex_int(node_insert_stmt, nt_col_properties_flags, n->node_properties_flags);
  sqlite3_bind_ex_blob(node_insert_stmt, nt_col_name, n->nodename, n->nodeNameLen, SQLITE_STATIC); // Node name
  sqlite3_bind_ex_blob(node_insert_stmt, nt_col_dsk, n->dsk, n->dskLen, SQLITE_STATIC);           // DSK
  sqlite3_bind_ex_blob(node_insert_stmt, nt_col_cc_versions, n->node_cc_versions, n->node_cc_versions_len, SQLITE_STATIC);
  sqlite3_bind_ex_int(node_insert_stmt, nt_col_node_is_zws_probed, n->node_is_zws_probed);
  sqlite3_step(node_insert_stmt);

  // Ghi endpoints
  for (e = list_head(n->endpoints); e; e = list_item_next(e)) {
    sqlite3_reset(ep_insert_stmt);
    sqlite3_bind_ex_int(ep_insert_stmt, et_col_endpointid, e->endpoint_id);
    sqlite3_bind_ex_int(ep_insert_stmt, et_col_nodeid, n->nodeid);
    sqlite3_bind_ex_blob(ep_insert_stmt, et_col_info, e->endpoint_info, e->endpoint_info_len, SQLITE_STATIC);     // CC list
    sqlite3_bind_ex_blob(ep_insert_stmt, et_col_aggr, e->endpoint_agg, e->endpoint_aggr_len, SQLITE_STATIC);      // Aggregations
    sqlite3_bind_ex_blob(ep_insert_stmt, et_col_name, e->endpoint_name, e->endpoint_name_len, SQLITE_STATIC);     // Endpoint name
    sqlite3_bind_ex_blob(ep_insert_stmt, et_col_location, e->endpoint_location, e->endpoint_loc_len, SQLITE_STATIC); // Endpoint location
    sqlite3_bind_ex_int(ep_insert_stmt, et_col_state, e->state);                                                  // Probe state ep
    sqlite3_bind_ex_int(ep_insert_stmt, et_col_installer_iconID, e->installer_iconID);                            // Icons
    sqlite3_bind_ex_int(ep_insert_stmt, et_col_user_iconID, e->user_iconID);
    sqlite3_step(ep_insert_stmt);
  }
}
```
**Giải thích dòng quan trọng:**
- `rd_data_store_nvm_free(n)`: bảo đảm không có dữ liệu cũ dư thừa → tránh bản ghi trùng.
- `sqlite3_bind_ex_int(... security_flags ...)`: persist snapshot hiện tại các cấp security đã probe / cấp phát.
- `node_version_cap_and_zwave_sw`: kết hợp Version Capability + Software info – phục vụ logic về version.
- `probe_flags`: phân biệt đã từng probe / probe fail để ra quyết định khi restart.
- Endpoint loop: mỗi endpoint lưu CC list riêng (`endpoint_info`) và metadata (tên, vị trí, icon) → dùng cho Multi Channel, mDNS, Naming CC.

### 3.2 Cập nhật node: `rd_data_store_update(n)`
```c
void rd_data_store_update(rd_node_database_entry_t *n) {
  // Cơ chế đơn giản: ghi lại toàn bộ node + endpoints
  rd_data_store_nvm_write(n);
}
```
> Không diff delta: luôn rewrite — đơn giản hóa logic; cần chú ý chi phí nếu nhiều updates liên tiếp.

### 3.3 Xóa node: `rd_data_store_nvm_free(n)`
```c
void rd_data_store_nvm_free(rd_node_database_entry_t *n) {
  // DELETE FROM nodes WHERE nodeid = ?
  // DELETE FROM endpoints WHERE nodeid = ?
  // Xóa sạch cả node lẫn endpoint đồng nhất
}
```
**Ý nghĩa:** bảo đảm nhất quán, không để orphan endpoints.

## 4. SPAN (S2 session) persistence

### 4.1 Ghi SPAN: `rd_datastore_persist_s2_span_table` (chỉ backup mục `SPAN_NEGOTIATED`)
```c
void rd_datastore_persist_s2_span_table(const struct SPAN *span_table, size_t sz) {
  datastore_exec_sql("DELETE FROM s2_span;");
  sqlite3_prepare_v2(db, "INSERT INTO s2_span VALUES(?,?,?,?,?,?,?)", ...);
  for (size_t i = 0; i < sz; i++) {
    const struct SPAN *entry = &span_table[i];
    if (entry->state == SPAN_NEGOTIATED) { // Chỉ persist phiên active
      sqlite3_reset(stmt);
      sqlite3_bind_ex_blob(stmt, span_col_d, &entry->d, sizeof(entry->d), SQLITE_STATIC); // D (ECDH shared / nonce seed)
      sqlite3_bind_ex_int(stmt,  span_col_lnode, entry->lnode);      // Local nodeid
      sqlite3_bind_ex_int(stmt,  span_col_rnode, entry->rnode);      // Remote nodeid
      sqlite3_bind_ex_int(stmt,  span_col_rx_seq, entry->rx_seq);    // RX seq
      sqlite3_bind_ex_int(stmt,  span_col_tx_seq, entry->tx_seq);    // TX seq
      sqlite3_bind_ex_int(stmt,  span_col_class_id, entry->class_id);// Security class
      sqlite3_bind_ex_int(stmt,  span_col_state, entry->state);      // State
      sqlite3_step(stmt);
    }
  }
}
```
**Giải thích:**
- Persist selective giảm kích thước DB & tránh rác session tạm.
- Các seq giữ để tránh duplication và tiếp tục dùng nonce/sequence logic sau restart.

### 4.2 Khôi phục SPAN: `rd_datastore_unpersist_s2_span_table`
```c
while (sqlite3_step(stmt) == SQLITE_ROW) {
  memcpy(&span_table[idx].d, sqlite3_column_blob(stmt, span_col_d), sizeof(span_table[idx].d));
  span_table[idx].lnode = sqlite3_column_int(stmt, span_col_lnode);
  span_table[idx].rnode = sqlite3_column_int(stmt, span_col_rnode);
  span_table[idx].rx_seq = sqlite3_column_int(stmt, span_col_rx_seq);
  span_table[idx].tx_seq = sqlite3_column_int(stmt, span_col_tx_seq);
  span_table[idx].class_id = sqlite3_column_int(stmt, span_col_class_id);
  span_table[idx].state   = sqlite3_column_int(stmt, span_col_state);
  idx++;
}
```
**Giải thích:** Khôi phục trạng thái đầy đủ để S2 tiếp tục mà không cần renegotiation ngay.

## 5. NAT Table (IPv4 mapping) – `ipv46_nat.c`

### 5.1 Thêm entry: `ipv46nat_add_entry(node)`
```c
u8_t ipv46nat_add_entry(nodeid_t node) {
  if (nat_table_size >= MAX_NAT_ENTRIES) return 0;           // Bảo vệ tràn
  for (i=0; i < nat_table_size; i++) if (nat_table[i].nodeid == node) return nat_table_size; // Tránh duplicate
  nat_table[nat_table_size].nodeid = node;                   // Gán NodeID
  nat_table[nat_table_size].ip_suffix =
        (node==MyNodeID) ? (uip_hostaddr.u16[1] & ~uip_netmask.u16[1]) : 0; // Gateway dùng suffix riêng, node khác chờ DHCP
  nat_table_size++;
  dhcp_check_for_new_sessions();                             // Kích hoạt cấp IP nếu cần
  return nat_table_size;
}
```
**Giải thích:**
- `ip_suffix` chứa phần host của địa chỉ IPv4; các node khác 0 → sẽ được DHCP lấp sau.
- Không persist: NAT bảng tái dựng dựa trên DHCP và node hiện có.

### 5.2 Lấy NodeID từ IPv4: `ipv46nat_get_nat_addr(ip)`
```c
nodeid_t ipv46nat_get_nat_addr(uip_ipv4addr_t *ip) {
  if (uip_ipaddr_maskcmp(ip, &uip_hostaddr, &uip_netmask)) {
    for (i=0; i < nat_table_size; i++) {
      if (nat_table[i].ip_suffix == (ip->u16[1] & (~uip_netmask.u16[1]))) return nat_table[i].nodeid;
    }
  }
  return 0;
}
```
**Giải thích:** So khớp subnet, sau đó map phần host → NodeID.

## 6. Keystore (Network Keys) – `S2_wrap.c`

### 6.1 Sinh key nếu thiếu: `keystore_network_generate_key_if_missing()`
```c
nvm_config_get(assigned_keys, &key_store_flags);
if ((key_store_flags & KEY_CLASS_S2_ACCESS) == 0) {
  S2_get_hw_random(net_key,16);
  keystore_network_key_write(KEY_CLASS_S2_ACCESS, net_key); // Persist key lớp ACCESS
}
// Tương tự cho AUTHENTICATED / UNAUTH / S0 / LR biến thể
```
**Giải thích:**
- `nvm_config_get/ set` truy cập vùng NVM cấu hình (ngoài SQLite). Mỗi bit KEY_CLASS_* đánh dấu key đã tồn tại.
- Khi tạo mới, key được ghi bằng `keystore_network_key_write(...)` để dùng cho mã hoá CCM (S2) / AES (S0).

## 7. Các nguồn dữ liệu khác liên quan node

| Thành phần | File | Cách lấy | Ghi chú |
|------------|------|----------|--------|
| Version CC probing state (`pcvs`) | `RD_internal.h` + logic trong `rd_ep_probe_cc_version` | Không persist | Runtime hỗ trợ state machine version |
| Portable flag (ROLE_TYPE_SLAVE_PORTABLE) | `rd_ep_zwave_plus_info_callback` | Persist vào `node_properties_flags` | Dùng cho wake-up logic |
| Downgrade security (mask bỏ S2/S0) | `rd_nif_request_notify` | Cập nhật `security_flags` trước persist | Bảo đảm không giữ key cấp cao nếu node không quảng cáo CC tương ứng |
| Failing status | `rd_set_failing` | Persist qua `rd_data_store_update` | Phân biệt node không phản hồi |
| Virtual nodes | `rd_datastore_persist_virtual_nodes` | SQLite bảng `virtual_nodes` | Forwarding qua bridge |
| IP Associations | `rd_data_store_persist_associations` | SQLite bảng `ip_association` | Mapping Group/Assoc host -> node |

## 8. Liên kết giữa security_flags và sử dụng

- Chọn scheme TX: `ZW_SendData_scheme_select` dùng `GetCacheEntryFlag(nodeid)` (dịch từ `security_flags` persist trong RD). 
- Kiểm tra unsolicited: `rd_check_security_for_unsolicited_dest` so sánh scheme nhận với “highest” mask.
- SPAN table + seq: chống duplication + tiếp tục phiên S2 sau restart.

## 9. Tóm tắt luồng cập nhật dữ liệu node khi inclusion / probe

1. Inclusion hoàn tất (S2 / Classic): Gán security flags (grant hoặc downgrade KNOWN_BAD).
2. Probe root: NIF → cập nhật `endpoint_info` root; có thể gỡ bỏ flag S0/S2 nếu CC SECURITY / SECURITY_2 không xuất hiện.
3. Manufacturer Specific → ghi manufacturerID/productType/productID.
4. Multi Channel enumeration → đếm endpoints + aggregated.
5. Endpoint capability → `endpoint_info` từng EP.
6. Z-Wave Plus Info → iconIDs + portable flag.
7. Version probing → điền `node_cc_versions`.
8. mDNS node / endpoint names → `nodename`, `endpoint_name`.
9. Persist tất cả: `rd_data_store_nvm_write(n)`.
10. Nếu S2 span active → persist SPAN table.

## 10. Điểm cần chú ý / Edge cases

- `rd_data_store_update` luôn ghi lại toàn bộ node + endpoints: tránh mismatch nhưng tăng I/O nếu gọi quá thường.
- SPAN persist chỉ lưu phiên ở trạng thái `SPAN_NEGOTIATED`: giảm rác, nhưng nếu mọi phiên ở trạng thái khác sẽ không khôi phục.
- NAT table không persist: sau restart có thể cần tái cấp phát IPv4 cho node để host IPv4 truy cập.
- Mã giảm cấp security (`rd_nif_request_notify`) phải chạy trước khi ghi để tránh cờ sai.
- Một số trường (pcvs) không persist – sau restart version probing có thể chạy lại nếu cần.

## 11. Checklist nhanh dữ liệu một node (RAM vs Persist)

| Field | RAM struct | Persist | Ghi bởi | Khi |
|-------|------------|---------|--------|-----|
| nodeid | rd_node_database_entry_t.nodeid | nodes.nodeid | rd_data_store_nvm_write | Add/Update |
| security_flags | rd_node_database_entry_t.security_flags | nodes.security_flags | rd_data_store_nvm_write | Sau probe / inclusion |
| manufacturerID/productType/productID | rd_node_database_entry_t | nodes.manufacturerID/... | rd_data_store_nvm_write | Manufacturer CC reply |
| wakeUp_interval | rd_node_database_entry_t.wakeUp_interval | nodes.wakeUp_interval | rd_data_store_nvm_write | WakeUp probe |
| lastAwake/lastUpdate | rd_node_database_entry_t | nodes.lastAwake/lastUpdate | rd_data_store_update | Khi nhận frame / probe finish |
| mode/state | rd_node_database_entry_t | nodes.mode/state | rd_data_store_nvm_write | Thay đổi FSM |
| probe_flags | rd_node_database_entry_t.probe_flags | nodes.probe_flags | rd_data_store_nvm_write | Probe milestone |
| node_properties_flags | rd_node_database_entry_t.node_properties_flags | nodes.properties_flags | rd_data_store_nvm_write | Z-Wave Plus / inclusion |
| nodename/nodeNameLen | rd_node_database_entry_t.nodename | nodes.name | rd_data_store_nvm_write | mDNS probe |
| dsk/dskLen | rd_node_database_entry_t.dsk | nodes.dsk | rd_data_store_nvm_write | Sau S2 auth |
| node_cc_versions + len | rd_node_database_entry_t.node_cc_versions | nodes.cc_versions | rd_data_store_nvm_write | Version probing |
| endpoint_info (+len) | rd_ep_database_entry_t.endpoint_info | endpoints.info | rd_data_store_nvm_write | Capability / NIF |
| endpoint_name/location (+len) | rd_ep_database_entry_t | endpoints.name/location | rd_data_store_nvm_write | mDNS / Naming CC |
| icons | rd_ep_database_entry_t.installer_iconID/user_iconID | endpoints.installer_iconID/user_iconID | rd_data_store_nvm_write | Z-Wave Plus Info |
| aggregated members | rd_ep_database_entry_t.endpoint_agg | endpoints.aggr | rd_data_store_nvm_write | Aggregated report |
| SPAN entries | s2_ctx->span_table | s2_span.* | rd_datastore_persist_s2_span_table | Khi persist spans |
| NAT IPv4 suffix | nat_table[i].ip_suffix | (Không) | ipv46nat_add_entry | Lúc tạo session/IP |

## 12. Kết luận
Z/IP Gateway lưu node data theo ba lớp chính: (1) RAM cấu trúc RD (node + endpoint + runtime), (2) SQLite persist (nodes / endpoints / spans / associations / virtual nodes), (3) NVM key store (network keys & config flags). NAT IPv4 mapping là bảng tạm trong RAM; SPAN bảng phiên bảo mật được persist chọn lọc. Việc ghi node dùng cơ chế “rewrite toàn bộ” để đơn giản hoá đồng bộ, chấp nhận chi phí I/O.

Nếu bạn muốn mở rộng thêm: có thể bổ sung sơ đồ sequence cho inclusion → probe → persist hoặc script kiểm tra nhất quán giữa RAM và DB.
