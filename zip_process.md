# ZIP Process - Central Orchestrator

## Overview

**`zip_process`** là **main orchestrator** của Z/IP Gateway - process trung tâm khởi động, quản lý và điều phối tất cả các process và service khác.

```c
// src/ZIP_Router.c:101
PROCESS(zip_process, "ZIP process");

// Main thread starts at line 1329
PROCESS_THREAD(zip_process, ev, data)
```

---

## 1. Vai trò của `zip_process`

### **Orchestrator - Điều phối viên trung tâm**
- Khởi động tất cả processes khác (TCP/IP stack, UDP server, DTLS, mDNS, Serial API, DHCP, etc.)
- Quản lý lifecycle: init → running → reset → shutdown
- Điều phối events giữa các subsystems (Network Management, Resource Directory, Bridge, Mailbox, etc.)
- Xử lý keyboard input và debug commands (khi chạy interactive mode)

### **State Machine Controller**
- Xử lý 15+ custom events (ZIP_EVENT_*)
- Quản lý các state transitions của gateway
- Đồng bộ các init phases (Reset → Bridge Init → All Probed → Network Management Done)

---

## 2. Process Dependencies - Cây phụ thuộc

### **`zip_process` khởi động các process sau:**

```
zip_process (main orchestrator)
    |
    ├── tcpip_process              ← IPv6 TCP/IP stack
    |
    ├── tcpip_ipv4_process         ← IPv4 TCP/IP stack (if portal enabled)
    |
    ├── udp_server_process         ← Z-Wave/IP UDP server (port 4123)
    |
    ├── dtls_server_process        ← DTLS encryption layer
    |
    ├── mDNS_server_process        ← mDNS responder (service discovery)
    |
    ├── serial_api_process         ← Z-Wave Serial API communication
    |
    ├── dhcp_client_process        ← DHCPv4 client (for IPv4 address)
    |
    ├── zip_tcp_client_process     ← Portal TCP tunnel (if configured)
    |
    ├── resolv_process             ← DNS resolver (if portal enabled)
    |
    └── ZW_SendDataAppl_process    ← Z-Wave TX queue (indirect via ZW_SendDataAppl_init)
```

### **Code locations - Khởi động processes:**
```c
// src/ZIP_Router.c:910-1048 (in ZIP_Router_Reset())

process_exit(&tcpip_process);
process_start(&tcpip_process, 0);               // Line 910

process_exit(&udp_server_process);
process_start(&udp_server_process, 0);          // Line 931

process_exit(&dtls_server_process);
process_start(&dtls_server_process, 0);         // Line 935

process_start(&mDNS_server_process, 0);         // Line 941 (no exit, only start once)

process_exit(&coap_server_process);
process_start(&coap_server_process, 0);         // Line 946

process_exit(&serial_api_process);
process_start(&serial_api_process, cfg.serial_port);  // Line 975

process_exit(&dhcp_client_process);
process_start(&dhcp_client_process, 0);         // Line 1048

// If portal enabled:
if (!cfg.ipv4disable && !ipv4_init_once) {
    process_start(&zip_tcp_client_process, cfg.portal_url);  // Line 1012
    process_start(&resolv_process, NULL);                    // Line 1016
}
```

---

## 3. Events Received by `zip_process`

### **Built-in Contiki Events:**
```c
PROCESS_EVENT_INIT          // Process khởi động lần đầu
PROCESS_EVENT_TIMER         // Timer expired
PROCESS_EVENT_EXITED        // Một process khác đã exit (VD: mDNS_server_process)
serial_line_event_message   // Serial input từ keyboard (debug mode)
```

### **Custom ZIP Gateway Events (15 events):**

| Event | Value | Sender | Purpose |
|-------|-------|--------|---------|
| `ZIP_EVENT_RESET` | (custom) | Self or NetworkManagement | Trigger full gateway reset |
| `ZIP_EVENT_TUNNEL_READY` | (custom) | Self (after TAP ready) | TAP tunnel up, networking ready |
| `ZIP_EVENT_BRIDGE_INITIALIZED` | (custom) | Bridge module | Bridge initialization complete |
| `ZIP_EVENT_ALL_NODES_PROBED` | (custom) | Resource Directory | All Z-Wave nodes interviewed |
| `ZIP_EVENT_NODE_PROBED` | (custom) | Resource Directory | Single node probed |
| `ZIP_EVENT_ALL_IPV4_ASSIGNED` | (custom) | DHCP client | All nodes got IPv4 addresses |
| `ZIP_EVENT_NODE_IPV4_ASSIGNED` | (custom) | DHCP client | Single node got IPv4 |
| `ZIP_EVENT_NODE_DHCP_TIMEOUT` | (custom) | DHCP client | DHCP timeout for some nodes |
| `ZIP_EVENT_NETWORK_MANAGEMENT_DONE` | (custom) | NetworkManagement | NM operation completed |
| `ZIP_EVENT_NM_VIRT_NODE_REMOVE_DONE` | (custom) | NetworkManagement | Virtual nodes removed |
| `ZIP_EVENT_QUEUE_UPDATED` | (custom) | Self or modules | Node command queue updated |
| `ZIP_EVENT_BACKUP_REQUEST` | (custom) | Self or external | Backup requested |
| `ZIP_EVENT_COMPONENT_DONE` | (custom) | ZGW components | Component initialization done |
| `resolv_event_found` | (from resolv_process) | DNS resolver | Hostname resolved (for portal) |
| `tcpip_event` | (from tcpip_process) | TCP/IP stack | Network data available |

---

## 4. Events Posted by `zip_process`

### **To Self:**
```c
process_post(&zip_process, ZIP_EVENT_RESET, 0);
process_post(&zip_process, ZIP_EVENT_QUEUE_UPDATED, 0);
process_post(&zip_process, ZIP_EVENT_TUNNEL_READY, 0);
```

### **To Other Processes:**
```c
// Broadcast exit to all processes
process_post(PROCESS_BROADCAST, PROCESS_EVENT_EXIT, 0);
```

### **Indirectly via Modules:**
Many events are posted by subsystems (NetworkManagement, ResourceDirectory, Bridge, DHCP) back to `zip_process`.

---

## 5. Main Event Loop - Flow Analysis

### **Event Handler Structure:**
```c
PROCESS_THREAD(zip_process, ev, data)
{
  PROCESS_BEGIN();
  
  while(1) {
    DBG_PRINTF("Event ***************** %x ********************\n", ev);
    
    // ===== EVENT HANDLERS =====
    
    if (ev == serial_line_event_message) {
      // Keyboard input (debug commands: l, d, r, b, x, etc.)
    }
    else if (ev == PROCESS_EVENT_INIT) {
      // First-time initialization
      if (!ZIP_Router_Reset()) {
        process_post(&zip_process, PROCESS_EVENT_EXIT, 0);
      }
    }
    else if (ev == ZIP_EVENT_RESET) {
      // Full gateway reset
      ZIP_Router_Reset();
    }
    else if (ev == ZIP_EVENT_TUNNEL_READY) {
      // TAP tunnel ready
      system_net_hook(1);  // Execute TunScript to add tap0 to br-lan
    }
    else if (ev == ZIP_EVENT_BRIDGE_INITIALIZED) {
      // Bridge ready
      ApplicationInitProtocols();
      NetworkManagement_Init();
    }
    else if (ev == ZIP_EVENT_ALL_NODES_PROBED) {
      // All Z-Wave nodes probed
      NetworkManagement_all_nodes_probed();
      send_nodelist();
    }
    else if (ev == ZIP_EVENT_NODE_IPV4_ASSIGNED) {
      // Node got IPv4 address
      NetworkManagement_IPv4_assigned(node);
    }
    else if (ev == ZIP_EVENT_ALL_IPV4_ASSIGNED) {
      // All nodes have IPv4
      udp_server_check_ipv4_queue();
      send_nodelist();
    }
    else if (ev == ZIP_EVENT_NETWORK_MANAGEMENT_DONE) {
      // NM operation complete
      send_nodelist();
      process_node_queues();
    }
    else if (ev == PROCESS_EVENT_TIMER) {
      // Timer expired (backup timer, etc.)
    }
    // ... more handlers ...
    
    PROCESS_WAIT_EVENT();  // Wait for next event
  }
  
  PROCESS_END();
}
```

---

## 6. Initialization Sequence - Detailed Flow

### **Boot Sequence:**

```
1. contiki-main() starts
   ↓
2. autostart_start() → zip_process started with PROCESS_EVENT_INIT
   ↓
3. zip_process receives PROCESS_EVENT_INIT
   ↓
4. ZIP_Router_Reset() called
   ↓
   ├─→ process_start(&serial_api_process)  (Z-Wave Serial API)
   ├─→ process_start(&tcpip_process)       (IPv6 stack)
   ├─→ process_start(&udp_server_process)  (Z-Wave/IP UDP)
   ├─→ process_start(&dtls_server_process) (DTLS encryption)
   ├─→ process_start(&mDNS_server_process) (mDNS responder)
   ├─→ process_start(&dhcp_client_process) (DHCPv4 client)
   └─→ If portal: process_start(&zip_tcp_client_process, &resolv_process)
   ↓
5. tcpip_process emits TCP_READY → BROADCAST
   ↓
6. zip_process posts ZIP_EVENT_TUNNEL_READY to self
   ↓
7. zip_process receives ZIP_EVENT_TUNNEL_READY
   → system_net_hook(1)  (exec TunScript: add tap0 to br-lan)
   ↓
8. bridge_init() completes → posts ZIP_EVENT_BRIDGE_INITIALIZED
   ↓
9. zip_process receives ZIP_EVENT_BRIDGE_INITIALIZED
   → ApplicationInitProtocols()
   → NetworkManagement_Init()
   ↓
10. Resource Directory probes all nodes
    ↓
11. rd_init() completes → posts ZIP_EVENT_ALL_NODES_PROBED
    ↓
12. zip_process receives ZIP_EVENT_ALL_NODES_PROBED
    → send_nodelist()
    ↓
13. DHCP assigns all IPv4 addresses → posts ZIP_EVENT_ALL_IPV4_ASSIGNED
    ↓
14. zip_process receives ZIP_EVENT_ALL_IPV4_ASSIGNED
    → NetworkManagement_NetworkUpdateStatusUpdate()
    → posts ZIP_EVENT_NETWORK_MANAGEMENT_DONE
    ↓
15. zip_process receives ZIP_EVENT_NETWORK_MANAGEMENT_DONE
    → Gateway fully initialized and ready!
```

---

## 7. Related Processes - Chi tiết từng process

### **7.1. `tcpip_process` - IPv6 Stack**
**Started by:** `zip_process` in `ZIP_Router_Reset()`  
**Purpose:** IPv6 TCP/IP stack  
**Events sent to zip_process:**
- `TCP_READY` (broadcast when stack ready)
- `tcpip_event` (if zip_process registers TCP/UDP connections)

**Events received from zip_process:** (none directly)

---

### **7.2. `udp_server_process` - Z-Wave/IP UDP Server**
**Started by:** `zip_process` in `ZIP_Router_Reset()`  
**Purpose:** Listen on port 4123 for Z-Wave/IP commands  
**Events sent to zip_process:** (none directly, but processes Z-Wave commands)  
**Events received from zip_process:**
- `TUNNEL_READY` (indirectly via broadcast)
- `tcpip_event` (when UDP packet arrives on port 4123)

**Connection:**
```c
// src/ZW_udp_server.c:1197
server_conn = udp_new(NULL, UIP_HTONS(0), &server_conn);
udp_bind(server_conn, UIP_HTONS(ZWAVE_PORT));  // ZWAVE_PORT = 4123

// When UDP packet arrives on port 4123:
// tcpip_process → tcpip_uipcall() → process_post_synch(udp_server_process, tcpip_event, &server_conn)
```

---

### **7.3. `dtls_server_process` - DTLS Encryption**
**Started by:** `zip_process` in `ZIP_Router_Reset()`  
**Purpose:** DTLS encryption layer for secure Z-Wave/IP  
**Events sent to zip_process:** (none)  
**Events received from zip_process:**
- `tcpip_event` (encrypted UDP packets)
- `PROCESS_EVENT_TIMER` (DTLS handshake timeouts)

**Connection:** Wraps `udp_server_process`, decrypts packets before forwarding.

---

### **7.4. `mDNS_server_process` - Service Discovery**
**Started by:** `zip_process` in `ZIP_Router_Reset()`  
**Purpose:** Announce Z/IP Gateway services on LAN (mDNS/DNS-SD)  
**Events sent to zip_process:** (none)  
**Events received from zip_process:**
- `tcpip_event` (mDNS queries on port 5353)
- `PROCESS_EVENT_TIMER` (periodic announcements)

**Connection:**
```c
// src/mDNSService.c:2522
server_conn = udp_new(0, UIP_HTONS(0), &server_conn);
udp_bind(server_conn, UIP_HTONS(MDNS_PORT));  // MDNS_PORT = 5353
uip_ds6_maddr_add(&mDNS_mcast_addr);  // Join multicast group
```

---

### **7.5. `serial_api_process` - Z-Wave Controller**
**Started by:** `zip_process` in `ZIP_Router_Reset()`  
**Purpose:** Communicate with Z-Wave controller via serial port  
**Events sent to zip_process:**
- `serial_line_event_message` (Z-Wave callbacks, notifications)

**Events received from zip_process:** (none directly)

**Connection:**
```c
// src/serial_api_process.c:19
PROCESS(serial_api_process, "ZW Serial api");

// Reads from /dev/ttyACM0 (or configured serial port)
// Posts serial_line_event_message to zip_process when Z-Wave events occur
```

---

### **7.6. `dhcp_client_process` - IPv4 Address Assignment**
**Started by:** `zip_process` in `ZIP_Router_Reset()`  
**Purpose:** Obtain IPv4 addresses for gateway and Z-Wave nodes  
**Events sent to zip_process:**
- `ZIP_EVENT_NODE_IPV4_ASSIGNED` (single node got IPv4)
- `ZIP_EVENT_ALL_IPV4_ASSIGNED` (all nodes have IPv4)
- `ZIP_EVENT_NODE_DHCP_TIMEOUT` (some nodes timed out)

**Events received from zip_process:**
- `tcpip_ipv4_event` (DHCP packets on port 68)
- `PROCESS_EVENT_TIMER` (DHCP retries)

---

### **7.7. `zip_tcp_client_process` - Portal Tunnel**
**Started by:** `zip_process` in `ZIP_Router_Reset()` (if portal configured)  
**Purpose:** IPv4 tunnel to Z-Wave cloud portal  
**Events sent to zip_process:** (none directly)  
**Events received from zip_process:**
- `tcpip_event` (TCP data from portal)
- `tcpip_ipv4_event` (IPv4 packets)
- `resolv_event_found` (portal hostname resolved)
- `PROCESS_EVENT_TIMER` (keepalive, reconnect)

---

### **7.8. `resolv_process` - DNS Resolver**
**Started by:** `zip_process` in `ZIP_Router_Reset()` (if portal configured)  
**Purpose:** Resolve portal hostname to IP address  
**Events sent to zip_process:**
- `resolv_event_found` (broadcast when DNS query completes)

**Events received from zip_process:**
- `tcpip_ipv4_event` (DNS replies on port 53)
- `PROCESS_EVENT_TIMER` (DNS query retries)

---

## 8. Event Flow Examples

### **Example 1: Gateway Boot**
```
contiki-main → zip_process (PROCESS_EVENT_INIT)
  ↓
zip_process → ZIP_Router_Reset()
  ↓
  → process_start(&tcpip_process)
  → process_start(&udp_server_process)
  → process_start(&serial_api_process)
  ...
  ↓
tcpip_process → BROADCAST (TCP_READY)
  ↓
zip_process → self (ZIP_EVENT_TUNNEL_READY)
  ↓
zip_process → system_net_hook(1)  [exec TunScript]
  ↓
bridge_init → zip_process (ZIP_EVENT_BRIDGE_INITIALIZED)
  ↓
zip_process → ApplicationInitProtocols() → NetworkManagement_Init()
  ↓
rd_init → zip_process (ZIP_EVENT_ALL_NODES_PROBED)
  ↓
zip_process → send_nodelist()
  ↓
dhcp_client → zip_process (ZIP_EVENT_ALL_IPV4_ASSIGNED)
  ↓
zip_process → NetworkManagement_NetworkUpdateStatusUpdate()
  ↓
NetworkManagement → zip_process (ZIP_EVENT_NETWORK_MANAGEMENT_DONE)
  ↓
GATEWAY READY!
```

### **Example 2: UDP Packet from LAN**
```
LAN host sends UDP to [gateway_ipv6]:4123
  ↓
br-lan → tap0 → tapdev_process
  ↓
tapdev_process → tcpip_process (PACKET_INPUT)
  ↓
tcpip_process → uip_input() parse packet
  ↓
tcpip_process → lookup UDP connection by port 4123
  ↓
tcpip_process → tcpip_uipcall() get conn->appstate.p = udp_server_process
  ↓
tcpip_process → udp_server_process (tcpip_event, &server_conn)
  ↓
udp_server_process → tcpip_handler() parse Z-Wave/IP command
  ↓
udp_server_process → forward to Z-Wave network via serial_api_process
```

### **Example 3: Gateway Reset**
```
User presses 'r' key (or NM receives SET_DEFAULT command)
  ↓
serial_line_process → zip_process (serial_line_event_message, "r")
  ↓
zip_process → self (ZIP_EVENT_RESET)
  ↓
zip_process → ZIP_Router_Reset()
  ↓
  → Stop all processes (process_exit)
  → Reinitialize everything
  → Start all processes (process_start)
  ↓
(Same init sequence as boot)
```

---

## 9. Process-Event Matrix for `zip_process`

| Event | Sender | Handler | Action |
|-------|--------|---------|--------|
| `PROCESS_EVENT_INIT` | Contiki (first start) | Line 1547 | Call `ZIP_Router_Reset()` |
| `serial_line_event_message` | `serial_line_process` | Line 1344 | Debug keyboard commands |
| `ZIP_EVENT_RESET` | Self or NM | Line 1555 | Full gateway reset |
| `ZIP_EVENT_TUNNEL_READY` | Self | Line 1565 | Execute `system_net_hook(1)` |
| `ZIP_EVENT_BRIDGE_INITIALIZED` | Bridge module | Line 1740 | Call `ApplicationInitProtocols()` |
| `ZIP_EVENT_ALL_NODES_PROBED` | Resource Directory | Line 1659 | Call `send_nodelist()` |
| `ZIP_EVENT_NODE_PROBED` | Resource Directory | Line 1677 | Notify NetworkManagement |
| `ZIP_EVENT_NODE_IPV4_ASSIGNED` | DHCP client | Line 1560 | Notify NetworkManagement |
| `ZIP_EVENT_ALL_IPV4_ASSIGNED` | DHCP client | Line 1681 | Run finish script, send nodelist |
| `ZIP_EVENT_NODE_DHCP_TIMEOUT` | DHCP client | Line 1681 | Same as ALL_IPV4_ASSIGNED |
| `ZIP_EVENT_NETWORK_MANAGEMENT_DONE` | NetworkManagement | Line 1756 | Send nodelist, process queues |
| `ZIP_EVENT_NM_VIRT_NODE_REMOVE_DONE` | NetworkManagement | Line 1772 | Notify virtual node removal |
| `ZIP_EVENT_QUEUE_UPDATED` | Self or modules | (implicit) | Process node command queues |
| `ZIP_EVENT_BACKUP_REQUEST` | Self or external | Line 1777 | Start backup procedure |
| `ZIP_EVENT_COMPONENT_DONE` | ZGW components | Line 1786 | Check backup completion |
| `PROCESS_EVENT_TIMER` | etimer_process | Line 1799 | Handle backup timer |
| `PROCESS_EVENT_EXITED` | mDNS or other | (not in snippet) | Handle process exit |
| `tcpip_event` | tcpip_process | (not used directly) | (zip_process doesn't use TCP/UDP) |
| `resolv_event_found` | resolv_process | (not in snippet) | Portal hostname resolved |

---

## 10. Summary

**`zip_process` is the heart of Z/IP Gateway:**

1. **Orchestrates** all other processes (15+ processes)
2. **Receives** 15+ custom ZIP_EVENT_* events
3. **Posts** events to self and broadcasts to subsystems
4. **Manages** gateway lifecycle: Init → Reset → Bridge → Probe → DHCP → Ready
5. **Does NOT** handle TCP/UDP directly (that's `udp_server_process`, `dtls_server_process`, etc.)
6. **Acts as** state machine controller and event router

**Key dependencies:**
- **Requires:** `tcpip_process`, `serial_api_process`, `udp_server_process`
- **Optional:** `dtls_server_process`, `mDNS_server_process`, `zip_tcp_client_process`, `resolv_process`
- **Drives:** NetworkManagement, ResourceDirectory, Bridge, Mailbox, ProvisioningList

**Most important events:**
- `ZIP_EVENT_RESET` → Full gateway reset
- `ZIP_EVENT_TUNNEL_READY` → TAP ready, execute TunScript
- `ZIP_EVENT_BRIDGE_INITIALIZED` → Bridge ready, start protocols
- `ZIP_EVENT_ALL_NODES_PROBED` → All Z-Wave nodes interviewed
- `ZIP_EVENT_NETWORK_MANAGEMENT_DONE` → Gateway fully operational
