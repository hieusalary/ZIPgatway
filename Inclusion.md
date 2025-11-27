# ZIP Gateway Inclusion Flow and Node Storage (S2/SmartStart)

This document traces the end-to-end flow when a node is included into the Z-Wave network via the Z/IP Gateway, and shows exactly what is stored for the node, where it is persisted, with code citations and short explanations.

## Overview

- Inclusion control and state: Network Management (NM) finite-state machine in `src/CC_NetworkManagement.c` with support from `src/transport/S2_wrap.c` for S2 events.
- Resource Directory (RD) registers the node and probes it, then persists everything to the SQLite datastore via `src/RD_DataStore_Sqlite.c`.
- Security: Gateway network keys live in keystore NVM (`src/transport/s2_keystore.c`). Node-specific “granted keys” are recorded as security flags in RD and used to select send schemes.

## Flow: From Inclusion Start to Persisted Node

### 1) Inclusion trigger and initial node info (NIF)

File: `src/ZW_ZIPApplication.c`

```c
// SmartStart inclusion trigger (example)
// ~ line 300
NetworkManagement_smart_start_inclusion(ADD_NODE_OPTION_NETWORK_WIDE | ADD_NODE_OPTION_NORMAL_POWER,
                                        prospectHomeID,
                                        /* test-prekit? */ false);

// INIF (initial NIF) received from protocol side
// ~ line 313
NetworkManagement_INIF_Received(bNodeID, INIF_rxStatus, INIF_NWI_homeid);
```

Explanation:
- The gateway requests inclusion (SmartStart or classic). When a node’s initial NIF is received, NM is notified to proceed.

### 2) S2 inclusion: key request, DSK challenge/accept, grant, completion

File: `src/transport/S2_wrap.c`

```c
// S2 inclusion event handler bridges S2 library events to NM
// ~ lines 140–207
static void sec2_event_handler(zwave_event_t* ev) {
  switch(ev->event_type) {
    case S2_NODE_INCLUSION_PUBLIC_KEY_CHALLENGE_EVENT:
      NetworkManagement_dsk_challenge(&(ev->evt.s2_event.s2_data.challenge_req));
      break;
    case S2_NODE_INCLUSION_KEX_REPORT_EVENT:
      NetworkManagement_key_request(&ev->evt.s2_event.s2_data.kex_report);
      break;
    case S2_NODE_INCLUSION_COMPLETE_EVENT:
      // Map exchanged keys to RD node flags and notify NM
      sec_incl_cb(keystore_flags_2_node_flags(
        ev->evt.s2_event.s2_data.inclusion_complete.exchanged_keys));
      break;
  }
}

// Gateway node flags derived from keystore assigned_keys
// ~ lines 268–279
uint8_t sec2_get_my_node_flags() {
  uint8_t flags;
  nvm_config_get(assigned_keys,&flags);
  return keystore_flags_2_node_flags(flags);
}
```

File: `src/CC_NetworkManagement.c`

```c
// In NM_WAIT_FOR_SECURE_ADD, upon S2 completion or stop
// ~ lines 900–1015
if (ev == NM_EV_SECURITY_DONE || ev == NM_NODE_ADD_STOP) {
  uint32_t inclusion_flags = (ev == NM_NODE_ADD_STOP)
    ? NODE_FLAG_KNOWN_BAD
    : (*(uint32_t*) event_data);

  // Persist node security flags (granted keys) to RD
  SetCacheEntryFlagMasked(nms.tmp_node, inclusion_flags & 0xFF, NODE_FLAGS_SECURITY);

  // If DSK is valid and S2 succeeded, add DSK to RD
  if (nms.dsk_valid == TRUE && (inclusion_flags & NODE_FLAGS_SECURITY2)
      && !(inclusion_flags & NODE_FLAG_KNOWN_BAD)) {
    rd_node_add_dsk(nms.tmp_node, 16, nms.just_included_dsk);
  }
}

// Handle key request from joining node, grant keys
// ~ lines 1001–1012
else if (ev == NM_EV_ADD_SECURITY_REQ_KEYS) {
  s2_node_inclusion_request_t *req = (s2_node_inclusion_request_t*) (event_data);
  uint8_t keys = req->security_keys; // may be intersected with provisioning list
  nms.granted_keys = keys;
  sec2_key_grant(NODE_ADD_KEYS_SET_EX_ACCEPT_BIT, keys, 0);
}

// Handle DSK challenge: accept DSK (possibly auto-filled from provisioning list)
// ~ lines 1043–1116
else if (ev == NM_EV_ADD_SECURITY_KEY_CHALLENGE) {
  s2_node_inclusion_challenge_t *challenge_evt = (s2_node_inclusion_challenge_t*) (event_data);
  sec2_dsk_accept(1, challenge_evt->public_key, 2);
  memcpy(nms.just_included_dsk, challenge_evt->public_key, sizeof(nms.just_included_dsk));
  nms.dsk_valid = TRUE;
}
```

Explanation:
- S2 events are forwarded to NM. After key exchange, NM records per-node security flags in RD (`SetCacheEntryFlagMasked`). If the DSK is authenticated, it’s stored with the node.
- Requested keys from the device are granted per policy/provisioning list and fed back to the S2 stack.

### 3) Transition to RD probe and neighbor update

File: `src/CC_NetworkManagement.c`

```c
// After secure add, move to probing
// ~ lines 980–1018
nms.state = NM_WAIT_FOR_PROBE_AFTER_ADD;
rd_probe_lock(FALSE);
```

Explanation:
- Once S2 is complete, the gateway unlocks RD probing to interview the node and collect detailed capabilities.

### 4) RD registers node and interviews endpoints/CCs

File: `src/ResourceDirectory.c`

```c
// Register new node and start probe
// ~ lines 2060–2135
void rd_register_new_node(nodeid_t node, uint8_t node_properties_flags) {
  rd_node_database_entry_t *n = rd_node_entry_alloc(node);
  // Endpoint 0 always exists
  rd_add_endpoint(n, 0);
  rd_node_probe_update(n); // drive probe state machine
}

// NIF received: update endpoint info; possibly downgrade security flags
// ~ lines 860–920
void rd_nif_request_notify(uint8_t bStatus, nodeid_t bNodeID, uint8_t* nif, uint8_t nif_len) {
  ep->endpoint_info = rd_data_mem_alloc(nif_len+1);
  memcpy(ep->endpoint_info, nif + 1, nif_len - 1);
  ep->endpoint_info[nif_len-1] = COMMAND_CLASS_ZIP_NAMING;
  ep->endpoint_info[nif_len  ] = COMMAND_CLASS_ZIP;
  ep->endpoint_info_len = nif_len+1;

  // If just added by this GW and root does not support CC Security/S2, downgrade flags
  if ((ep->endpoint_id == 0)
      && !(ep->node->node_properties_flags & RD_NODE_FLAG_ADDED_BY_ME)
      && (ep->node->node_properties_flags & RD_NODE_FLAG_JUST_ADDED)) {
    if (!rd_ep_supports_cmd_class_nonsec(ep, COMMAND_CLASS_SECURITY_2)) {
      ep->node->security_flags &= ~NODE_FLAGS_SECURITY2;
    }
    if (!rd_ep_supports_cmd_class_nonsec(ep, COMMAND_CLASS_SECURITY)) {
      ep->node->security_flags &= ~NODE_FLAG_SECURITY0;
    }
  }
}

// Probe secure commands per scheme
// ~ lines 1180–1280
// Example for ACCESS key class on root endpoint
if (GetCacheEntryFlag(MyNodeID) & NODE_FLAG_SECURITY2_ACCESS) {
  p.scheme = SECURITY_SCHEME_2_ACCESS;
  ZW_SendRequest(&p, secure_commands_supported_get2, sizeof(secure_commands_supported_get2),
                 SECURITY_2_COMMANDS_SUPPORTED_REPORT, 20, ep, rd_ep_secure_commands_get_callback);
}
```

Explanation:
- RD sets up the node entry, endpoint 0, and starts the probe machine to collect NIF/CCs. If the node’s actual NIF does not advertise Security/S2, RD lowers the earlier security flags to match reality.
- RD queries S2/S0 `COMMANDS_SUPPORTED` using a scheme determined by both the node’s flags and the gateway’s own flags.

### 5) Probe completes and node is persisted

File: `src/ResourceDirectory.c`

```c
// On STATUS_DONE or STATUS_PROBE_FAIL, persist in SQLite
// ~ lines 2008–2040
rd_data_store_nvm_free(n);
rd_data_store_nvm_write(n);
```

File: `src/RD_DataStore_Sqlite.c`

```c
// Write node and all endpoints to SQLite
// ~ lines 560–760
void rd_data_store_nvm_write(rd_node_database_entry_t *n) {
  // nodes table: nodeid, security_flags, mode, state, manufacturerID, productType, productID,
  // nodename, dsk, cc_versions, timestamps, etc.
  // endpoints table: endpoint info, name, location, icon IDs, state
}

// Read node back from SQLite (used at startup)
// ~ lines 420–560
rd_node_database_entry_t *rd_data_store_read(nodeid_t nodeID) {
  // fills n->security_flags, n->dsk, n->node_cc_versions, and builds list of endpoints
}
```

Explanation:
- RD persists all node and endpoint data to SQLite, so the node’s state survives process restarts.

### 6) Persist S2 SPAN table (optional, if negotiated)

File: `src/transport/S2_wrap.c`

```c
// ~ lines 852–868
void sec2_persist_span_table() {
  rd_datastore_persist_s2_span_table(s2_ctx->span_table, SPAN_TABLE_SIZE);
}

// ~ lines 872–900
void sec2_unpersist_span_table() {
  rd_datastore_unpersist_s2_span_table(s2_ctx->span_table, SPAN_TABLE_SIZE);
}
```

File: `src/RD_DataStore_Sqlite.c`

```c
// ~ lines 967–992
void rd_datastore_persist_s2_span_table(const struct SPAN *span_table, size_t span_table_size) {
  // store entries with state == SPAN_NEGOTIATED into s2_span table
}

// ~ lines 995–1026
void rd_datastore_unpersist_s2_span_table(struct SPAN *span_table, size_t span_table_size) {
  // restore negotiated spans at startup
}
```

Explanation:
- If the SPAN (secure channel state) is negotiated, entries are saved/restored to avoid re-handshaking every time the gateway restarts.

## What is Stored for a Node (Summary Table)

| Category | Field(s) | Runtime Struct | Persisted In | Writer (file/function) | When |
|---|---|---|---|---|---|
| Security flags (granted keys) | `n->security_flags` | `rd_node_database_entry_t` | SQLite `nodes.security_flags` | `CC_NetworkManagement.c` → `SetCacheEntryFlagMasked(...)` which calls `rd_data_store_update(n)` | After S2 completion (`NM_EV_SECURITY_DONE`) |
| DSK | `n->dsk`, `n->dskLen` | `rd_node_database_entry_t` | SQLite `nodes.dsk` | `CC_NetworkManagement.c` → `rd_node_add_dsk(...)`; `RD_internal.c` implements | After DSK authenticated |
| NIF & CC list per endpoint | `ep->endpoint_info`, `endpoint_info_len` | `rd_ep_database_entry_t` | SQLite `endpoints.info` | `ResourceDirectory.c` → `rd_nif_request_notify`, `rd_ep_probe_update` | During RD probe |
| Manufacturer/Product | `n->manufacturerID`, `productType`, `productID` | `rd_node_database_entry_t` | SQLite `nodes.*` | `ResourceDirectory.c` → `rd_probe_vendor_callback` | During RD probe |
| WakeUp interval & timestamps | `n->wakeUp_interval`, `lastAwake`, `lastUpdate` | `rd_node_database_entry_t` | SQLite `nodes.*` | `ResourceDirectory.c` probe states | During RD probe |
| Endpoint name/location | `ep->endpoint_name`, `endpoint_location` | `rd_ep_database_entry_t` | SQLite `endpoints.*` | `ResourceDirectory.c` mdns probe; `RD_internal.c` via DSK provisioning TLVs | RD mdns probe or DSK provisioning |
| SPAN S2 (if negotiated) | `SPAN` entries | S2 context | SQLite `s2_span` | `S2_wrap.c` → persist/unpersist; `RD_DataStore_Sqlite.c` | At S2 negotiation / startup |
| Gateway keys (S2/S0) | `security_netkey`, `security2_key[]`, `assigned_keys` | Keystore NVM | NVM (not per-node) | `s2_keystore.c` | Device provisioning / init |

## Send Scheme Selection (How the flags are used later)

File: `src/transport/ZW_SendDataAppl.c`

```c
// ~ lines 188–246
static security_scheme_t ZW_SendData_scheme_select(const ts_param_t* param, const u8_t* data, u8_t len) {
  u8_t dst_scheme_mask = GetCacheEntryFlag(param->dnode);
  u8_t src_scheme_mask = GetCacheEntryFlag(MyNodeID);
  // choose the highest mutual scheme; CRC16 fallback if needed
}
```

Explanation:
- When sending commands post-inclusion, the gateway uses the node’s stored security flags (plus its own) to pick the appropriate security scheme (S2 Access/Auth/Unauth, S0, CRC16, or none).

## Notes

- Gateway network keys are never stored per node and never sent on the wire. Only the node’s granted key classes are recorded as flags.
- RD may adjust security flags if the node’s actual reported NIF does not include CC Security/S2.
- The inclusion flow for Long Range (LR) nodes uses 16-bit node IDs in S2 AAD and status frames; the storage/persist mechanism remains the same.
