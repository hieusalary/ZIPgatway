# ZIP_PROCESS - Trung Tâm Điều Phối ZIP Gateway

## Tổng quan

`zip_process` là **MAIN CONTROL PROCESS** của ZIP Gateway - đóng vai trò như một **Event-Driven State Machine** quản lý toàn bộ lifecycle và hoạt động của gateway.

**Location**: `src/ZIP_Router.c:1342`
**Type**: Contiki PROCESS (event-driven thread)
**Auto-start**: Được start tự động qua `AUTOSTART_PROCESSES(&zip_process)`

---

## Kiến trúc Event-Driven

```c
PROCESS_THREAD(zip_process, ev, data)
{
  PROCESS_BEGIN();
  
  while(1) {
    // Process các events
    switch(ev) {
      case ZIP_EVENT_RESET: ...
      case ZIP_EVENT_TUNNEL_READY: ...
      case ZIP_EVENT_QUEUE_UPDATED: ...
      // ... nhiều events khác
    }
    
    PROCESS_WAIT_EVENT();  // Block cho đến khi có event mới
  }
  
  PROCESS_END();
}
```

---

## Các Chức Năng Chính (21 Event Handlers)

### **1. PROCESS_EVENT_INIT** - Khởi động Gateway
**Khi nào**: Gateway vừa start lần đầu  
**Làm gì**:
```c
if (!ZIP_Router_Reset()) {
  ERR_PRINTF("Fatal error\n");
  process_post(&zip_process, PROCESS_EVENT_EXIT, 0);
}
```

**Chi tiết**:
- Gọi `ZIP_Router_Reset()` để init tất cả subsystems
- Load configuration từ file
- Init Z-Wave controller
- Setup network interfaces
- Nếu fail → Exit process

**Ví dụ flow**:
```
[Startup] 
  → PROCESS_EVENT_INIT
  → ZIP_Router_Reset()
    ├─ Read config file
    ├─ Init Z-Wave NVM
    ├─ Setup IPv6 addresses
    ├─ Init DTLS server
    └─ Start mDNS service
  → Success → Wait for next event
```

---

### **2. ZIP_EVENT_RESET** - Reset Gateway
**Khi nào**: Được gọi thủ công (keyboard 'r') hoặc sau network changes  
**Làm gì**:
```c
if (data == (void*) 0) {
  LOG_PRINTF("Resetting....\n");
  zgw_component_start(ZGW_ROUTER_RESET);
  if (!ZIP_Router_Reset()) {
    ERR_PRINTF("Fatal error\n");
    process_post(&zip_process, PROCESS_EVENT_EXIT, 0);
  }
}
```

**Trigger từ**:
- User nhấn 'r' keyboard
- `CC_NetworkManagement.c:1262` - Sau khi learn mode
- `ZW_ZIPApplication.c:372` - Sau SmartStart inclusion

**Ví dụ**:
```
[User presses 'r']
  → ZIP_EVENT_RESET posted
  → zip_process receives event
  → Mark component ZGW_ROUTER_RESET active
  → ZIP_Router_Reset()
    ├─ Re-read config
    ├─ Refresh Z-Wave network info
    ├─ Rebuild routing tables
    └─ Restart discovery
  → Gateway operational again
```

---

### **3. ZIP_EVENT_TUNNEL_READY** - Portal Tunnel Established
**Khi nào**: SSL tunnel đến portal service được thiết lập thành công  
**Làm gì**:
```c
LOG_PRINTF("[HIEU] TUNNEL READY!\n");

// Refresh used lan and pan addresses
ZIP_eeprom_init();

// Reinit the IPv6 addresses
refresh_ipv6_addresses();

// Bring network interface up
system_net_hook(1);

// Try to send pending NM replies
NetworkManagement_Init();
```

**Trigger từ**: `ZW_tcp_client.c:276` khi SSL handshake hoàn thành

**Ví dụ**:
```
[TCP Client connects to portal]
  → SSL handshake succeeds
  → ZIP_EVENT_TUNNEL_READY posted
  → zip_process handles:
    1. Reload IPv6 addresses from EEPROM
    2. Call system_net_hook(1) → Execute network script
    3. Enable routing to portal
    4. Allow NetworkManagement to send replies
    
[Result]: Gateway can now forward packets to/from portal
```

---

### **4. tcpip_event + TCP_READY** - Network Stack Ready
**Khi nào**: Contiki TCP/IP stack initialization hoàn thành  
**Làm gì**:
```c
if (data == (void*) TCP_READY) {
  LOG_PRINTF("[HIEU] Comming up\n");
  system_net_hook(1);           // Bring network up
  
  ClassicZIPNode_init();        // Init Z-Wave classic node support
  zgw_component_start(ZGW_BRIDGE);
  bridge_init();                // Start bridge initialization
}
```

**Ví dụ**:
```
[Contiki TCP/IP initialized]
  → tcpip_event with TCP_READY posted
  → zip_process:
    1. Execute network script (setup TAP, bridge, routes)
    2. Init ClassicZIPNode (for backward compatibility)
    3. Start bridge component
    4. bridge_init() → Create virtual nodes
    
[Result]: Network interfaces br-lan, tap0 ready
```

---

### **5. ZIP_EVENT_BRIDGE_INITIALIZED** - Bridge Init Complete
**Khi nào**: Virtual nodes created, bridge ready  
**Làm gì**:
```c
LOG_PRINTF("Bridge init done\n");
zgw_component_done(ZGW_BRIDGE, data);

ApplicationInitProtocols();     // Start Z/IP protocols
sec2_unpersist_span_table();   // Restore S2 keys
NetworkManagement_Init();
send_nodelist();                // Send node list to portal

tcpip_ipv4_force_start_periodic_tcp_timer();
NetworkManagement_NetworkUpdateStatusUpdate(0);
```

**Trigger từ**:
- `Bridge.c:159` - Normal bridge init done
- `Bridge_temp_assoc.c:289` - Temp association done

**Ví dụ**:
```
[Bridge creates virtual nodes]
  → ZIP_EVENT_BRIDGE_INITIALIZED posted
  → zip_process:
    1. Start Z/IP Application protocols
       - Command handlers (Basic, Switch, etc)
       - Firmware Update
       - Security S0/S2
    2. Restore S2 SPAN table from persistence
    3. Enable Network Management
    4. Send node list to portal (UNSOLICITED report)
    5. Start IPv4 timers for DHCP
    
[Result]: Gateway fully operational, can handle Z/IP commands
```

---

### **6. ZIP_EVENT_QUEUE_UPDATED** - Node Queue Changed
**Khi nào**: Có node mới cần xử lý trong queue  
**Làm gì**:
```c
process_node_queues();
```

**Chi tiết**:
- Node queue chứa các nodes pending commands
- `process_node_queues()` iterate qua queue và send commands
- Được trigger từ nhiều nơi:
  - `node_queue.c:198` - Node added to queue
  - `node_queue.c:442` - After send completion
  - `Mailbox.c:931` - Mailbox has messages

**Ví dụ**:
```
[Controller sends BASIC_SET to Node 5]
  → Command queued in node_queue
  → ZIP_EVENT_QUEUE_UPDATED posted
  → zip_process:
    ├─ process_node_queues()
    ├─ Check if Node 5 is ready
    ├─ If ready: Send command via Z-Wave RF
    └─ If busy: Keep in queue
    
[Node 5 replies]
  → ZIP_EVENT_QUEUE_UPDATED posted again
  → Process next command in queue
```

---

### **7. ZIP_EVENT_ALL_NODES_PROBED** - Discovery Complete
**Khi nào**: Resource Directory đã probe xong tất cả nodes  
**Làm gì**:
```c
LOG_PRINTF("All nodes have been probed\n");
NetworkManagement_all_nodes_probed();
NetworkManagement_NetworkUpdateStatusUpdate(NETWORK_UPDATE_FLAG_PROBE);

NetworkManagement_smart_start_init_if_pending();

// Send node list if all nodes have IP
if(ipv46nat_all_nodes_has_ip()) {
  send_nodelist();
}

// Restart node queue
process_post(&zip_process, ZIP_EVENT_QUEUE_UPDATED, 0);

// Check if backup needed
zip_router_check_backup(data);
```

**Trigger từ**: `ResourceDirectory.c:1472` sau khi probe xong all nodes

**Ví dụ**:
```
[Gateway starts up with 5 nodes]
  → Resource Directory probes each node:
    Node 1: Probe → Get NIF → Get version → Get S2 keys
    Node 2: Probe → Get NIF → Get version → Get S2 keys
    ... (nodes 3-5)
    
[All probes complete]
  → ZIP_EVENT_ALL_NODES_PROBED posted
  → zip_process:
    1. Mark probe phase complete
    2. Update network status flags
    3. Enable SmartStart (if was disabled during probe)
    4. If all nodes have IPv4 → send_nodelist()
    5. Restart queue processing
    6. Check if backup requested
    
[Result]: 
- Gateway knows capabilities of all nodes
- Can send node list to portal
- Ready for normal operation
```

---

### **8. ZIP_EVENT_NODE_PROBED** - Single Node Probed
**Khi nào**: Một node cụ thể probe xong  
**Làm gì**:
```c
LOG_PRINTF("ZIP_EVENT_NODE_PROBED\n");
NetworkManagement_node_probed(data);  // data = rd_node_database_entry_t*
```

**Trigger từ**: `ResourceDirectory.c:2026` sau mỗi node probe

**Ví dụ**:
```
[Node 7 just included into network]
  → Resource Directory starts probe:
    1. Send NIF request
    2. Get Command Classes
    3. Get version info
    4. Get S2 keys (if secure)
    
[Probe completes]
  → ZIP_EVENT_NODE_PROBED posted with node=7
  → zip_process → NetworkManagement_node_probed(node 7)
    ├─ Update NM state machine
    ├─ If was waiting for this node → continue NM operation
    └─ Mark node as interviewed
```

---

### **9. ZIP_EVENT_ALL_IPV4_ASSIGNED** - DHCP Complete
**Khi nào**: DHCP đã cấp IPv4 cho tất cả nodes (hoặc timeout)  
**Làm gì**:
```c
LOG_PRINTF("ZIP_EVENT_ALL_IPV4_ASSIGNED\n");

// Execute external script with gateway's IPv4
char ip[IPV6_STR_LEN];
snprintf(ip, IPV6_STR_LEN, "IP=%d.%d.%d.%d", 
         uip_hostaddr.u8[0], uip_hostaddr.u8[1], 
         uip_hostaddr.u8[2], uip_hostaddr.u8[3]);

fork();
execve(linux_conf_fin_script, args, env);  // Run script

udp_server_check_ipv4_queue();  // Check pending IPv4 packets

if(!rd_probe_in_progress()) {
  send_nodelist();
}

NetworkManagement_NetworkUpdateStatusUpdate(NETWORK_UPDATE_FLAG_DHCPv4);
```

**Trigger từ**: `dhcpc2.c:498` khi DHCP round hoàn thành

**Ví dụ**:
```
[DHCP client requests IPs for nodes]
  Node 1: 192.168.1.100 ← DHCP ACK
  Node 2: 192.168.1.101 ← DHCP ACK
  Node 3: timeout (no DHCP server response)
  Node 4: 192.168.1.102 ← DHCP ACK
  
[DHCP round completes]
  → ZIP_EVENT_ALL_IPV4_ASSIGNED posted
  → zip_process:
    1. Fork child process
    2. Execute zipgateway.fin script:
       - Script receives IP=192.168.1.50 (gateway IP)
       - Can setup firewall rules
       - Can update DNS
    3. Check IPv4 packet queue (send pending packets)
    4. If no nodes being probed → send node list
    5. Update network flags
    
[Result]: 
- External script notified
- IPv4 routing active
- Can send IPv4-mapped packets
```

---

### **10. ZIP_EVENT_NODE_IPV4_ASSIGNED** - Single Node Got IP
**Khi nào**: Một node nhận được IPv4 address  
**Làm gì**:
```c
nodeid_t node = ((uintptr_t)data) & 0xFFFF;
LOG_PRINTF("New IPv4 assigned for node %d\n", node);

if (node == MyNodeID) {
  // Gateway itself got IPv4
  NetworkManagement_Init();  // Can now route IPv4 to gateway
} else {
  // Another node got IPv4
  NetworkManagement_IPv4_assigned(node);
}
```

**Ví dụ**:
```
[DHCP ACK for Node 5]
  → Node 5 assigned 192.168.1.105
  → ZIP_EVENT_NODE_IPV4_ASSIGNED posted with node=5
  → zip_process:
    ├─ Check if node == MyNodeID?
    │  └─ No, it's node 5
    ├─ Call NetworkManagement_IPv4_assigned(5)
    │  └─ If NM was waiting for this → continue operation
    └─ Gateway can now route to ::ffff:192.168.1.105
    
[Client sends packet to ::ffff:192.168.1.105]
  → Gateway forwards to Node 5 via Z-Wave
```

---

### **11. ZIP_EVENT_NETWORK_MANAGEMENT_DONE** - NM Operation Complete
**Khi nào**: Network Management hoàn thành một operation (learn, add, remove node)  
**Làm gì**:
```c
LOG_PRINTF("ZIP_EVENT_NETWORK_MANAGEMENT_DONE\n");

// If gateway just started, mark done
if (zgw_initing == TRUE) {
  set_should_send_nodelist();
  zgw_component_done(ZGW_BOOTING, data);
  zgw_initing = FALSE;
}

send_nodelist();
process_node_queues();

if (ZGW_COMPONENT_ACTIVE(ZGW_ROUTER_RESET)) {
  zgw_component_done(ZGW_ROUTER_RESET, data);
}

if (NetworkManagement_idle()) {
  zgw_component_done(ZGW_NM, data);
}
```

**Trigger từ**: `CC_NetworkManagement.c:1800` khi NM state → NM_IDLE

**Ví dụ**:
```
[User adds new node via controller]
  1. Controller sends NODE_ADD command
  2. Gateway enters learn mode
  3. New device included
  4. Gateway interviews device
  5. NM state machine → NM_IDLE
  
[NM Done]
  → ZIP_EVENT_NETWORK_MANAGEMENT_DONE posted
  → zip_process:
    ├─ If first time (zgw_initing) → Mark booting done
    ├─ send_nodelist() → Update portal with new node
    ├─ process_node_queues() → Send pending commands
    ├─ If was in reset → Mark reset done
    └─ Mark NM component idle
    
[Result]: Portal knows about new node, can send commands to it
```

---

### **12. ZIP_EVENT_NEW_NETWORK** - Entered New Z-Wave Network
**Khi nào**: Gateway learn vào network mới (hoặc được reset)  
**Làm gì**:
```c
// Generate new ULA prefix for PAN
new_pan_ula();
```

**Chi tiết**:
```c
static void new_pan_ula() {
  uip_ip6addr_t pan_prefix;
  LOG_PRINTF("Entering new network\n");
  create_ula_prefix(&pan_prefix, 2);  // Create random ULA
  
  // Save to NVM
  zw_appl_nvm_write(offsetof(zip_nvm_config_t, ula_pan_prefix), 
                    &pan_prefix, sizeof(pan_prefix));
}
```

**Trigger từ**: `CC_NetworkManagement.c:1259` và `2298`

**Ví dụ**:
```
[Gateway learns into new network]
  Old network: HomeID=0xAABBCCDD, NodeID=1
  New network: HomeID=0x11223344, NodeID=5
  
[Learn complete]
  → ZIP_EVENT_NEW_NETWORK posted
  → zip_process → new_pan_ula()
    ├─ Generate new ULA prefix: fd00:bbbb::/64
    ├─ Save to NVM
    └─ Will be used for Z-Wave nodes
    
[Result]: 
- Old nodes unreachable (different HomeID)
- New ULA prefix for new network
- Gateway resets with new identity
```

---

### **13. ZIP_EVENT_COMPONENT_DONE** - Component Finished Task
**Khi nào**: Một component (Mailbox, PowerLevel, NodeQueue) hoàn thành task  
**Làm gì**:
```c
LOG_PRINTF("ZIP_EVENT_COMPONENT_DONE\n");

if (ZGW_COMPONENT_ACTIVE(ZGW_BU)) {
  // If backup was requested, check if can start
  if (data == (void*)extend_middleware_probe_timeout) {
    DBG_PRINTF("NetworkManagement done after middleware probe timeout.\n");
  }
}

zip_router_check_backup(data);
```

**Trigger từ**:
- `node_queue.c:515` - Node queue idle
- `CC_NetworkManagement.c:3383` - NM idle after probe timeout
- `Bridge_ip_assoc.c:863` - Bridge association done

**Ví dụ**:
```
[Backup requested via SIGUSR1]
  → ZIP_EVENT_BACKUP_REQUEST posted
  → Mark ZGW_BU component active
  → Wait for all components to be idle
  
[Node queue finishes last command]
  → ZIP_EVENT_COMPONENT_DONE posted
  → zip_process → zip_router_check_backup()
    ├─ Check if all components idle?
    │  ├─ Node queue: ✓ idle
    │  ├─ Mailbox: ✓ idle
    │  ├─ PowerLevel: ✓ idle
    │  └─ Network Management: ✓ idle
    ├─ All idle! → Start backup
    └─ Call backup_and_overwrite_nvm()
    
[Result]: NVM backed up safely when nothing in progress
```

---

### **14. ZIP_EVENT_BACKUP_REQUEST** - Backup Requested
**Khi nào**: SIGUSR1 signal received  
**Làm gì**:
```c
LOG_PRINTF("ZIP_EVENT_BACKUP_REQUEST\n");

if (ZGW_COMPONENT_ACTIVE(ZGW_BU)) {
  WRN_PRINTF("Backup already requested once.\n");
}

zgw_component_start(ZGW_BU);
zip_router_check_backup(data);
```

**Ví dụ**:
```
[Admin sends signal]
  $ kill -SIGUSR1 $(pidof zipgateway)
  
[Signal handler]
  → process_post(&zip_process, ZIP_EVENT_BACKUP_REQUEST, NULL)
  
[zip_process receives event]
  → Mark backup component active
  → zip_router_check_backup()
    ├─ Check if gateway idle?
    │  └─ No, node queue has pending commands
    ├─ Set flag: backup_pending = TRUE
    └─ Wait for ZIP_EVENT_COMPONENT_DONE
    
[Later when idle]
  → ZIP_EVENT_COMPONENT_DONE
  → zip_router_check_backup()
    ├─ backup_pending = TRUE
    ├─ All components idle ✓
    └─ Execute backup NOW
```

---

### **15. ZIP_EVENT_NM_VIRT_NODE_REMOVE_DONE** - Virtual Node Removed
**Khi nào**: Virtual node association đã được xóa  
**Làm gì**:
```c
DBG_PRINTF("ZIP_EVENT_NM_VIRT_NODE_REMOVE_DONE triggered\n");
NetworkManagement_VirtualNodes_removed();
```

**Trigger từ**: `Bridge_ip_assoc.c:214` và `1107`

**Ví dụ**:
```
[Controller sends NODE_REMOVE for node 10]
  1. Gateway removes node from Z-Wave network
  2. Remove virtual node associations
  3. Remove IP associations
  
[Cleanup complete]
  → ZIP_EVENT_NM_VIRT_NODE_REMOVE_DONE posted
  → zip_process → NetworkManagement_VirtualNodes_removed()
    └─ NM can now send NODE_REMOVE reply
    
[Result]: Controller notified that removal complete
```

---

### **16. PROCESS_EVENT_EXITED** - Subprocess Terminated
**Khi nào**: Một process con exit  
**Làm gì**:
```c
LOG_PRINTF("A process exited %p\n", data);

if (data == &dtls_server_process) {
  LOG_PRINTF("dtls process exited\n");
  dtls_close_all();
} else if (data == &mDNS_server_process) {
  LOG_PRINTF("mDNS process exited\n");
  sec2_persist_span_table();    // Save S2 keys
  rd_destroy();                  // Destroy resource directory
  NetworkManagement_mdns_exited();
  data_store_exit();             // Close database
}
```

**Ví dụ**:
```
[Gateway shutdown sequence]
  1. User sends SIGTERM
  2. Main process starts shutdown
  3. DTLS server process exits
  
[DTLS exits]
  → PROCESS_EVENT_EXITED posted with data=&dtls_server_process
  → zip_process:
    └─ dtls_close_all() → Close all DTLS sessions
    
  4. mDNS server process exits
  
[mDNS exits]
  → PROCESS_EVENT_EXITED posted with data=&mDNS_server_process
  → zip_process:
    ├─ sec2_persist_span_table() → Save S2 keys to disk
    ├─ rd_destroy() → Clean up resource directory
    ├─ NetworkManagement_mdns_exited()
    └─ data_store_exit() → Close SQLite database
    
[Result]: Clean shutdown with data persistence
```

---

### **17. PROCESS_EVENT_EXIT** - Gateway Shutting Down
**Khi nào**: Gateway nhận lệnh tắt  
**Làm gì**:
```c
LOG_PRINTF("Bye bye\n");
zgw_component_start(ZGW_SHUTTING_DOWN);

#ifdef NO_ZW_NVM
zw_appl_nvm_close();
#endif
```

**Ví dụ**:
```
[User presses 'x' or sends SIGTERM]
  → process_post(&zip_process, PROCESS_EVENT_EXIT, 0)
  
[zip_process receives EXIT]
  → Mark shutting down state
  → Close Z-Wave NVM
  → Process broadcasts PROCESS_EVENT_EXIT
    ├─ All child processes receive exit signal
    ├─ DTLS server shuts down
    ├─ mDNS server shuts down
    ├─ UDP server shuts down
    └─ TCP client disconnects
    
[All processes exited]
  → Main loop exits
  → zipgateway process terminates
```

---

### **18. PROCESS_EVENT_TIMER** - Timer Expired
**Khi nào**: Timer được set trước đó timeout  
**Làm gì**:
```c
LOG_PRINTF("PROCESS_EVENT_TIMER\n");

if (data == (void*)&zgw_bu_timer) {
  // Backup timer expired
  zip_router_check_backup(data);
}
```

**Ví dụ**:
```
[Backup requested but gateway busy]
  → Set timer: zgw_bu_timer for 5 minutes
  → Wait for timeout OR components to be idle
  
[5 minutes later, still busy]
  → PROCESS_EVENT_TIMER fired
  → zip_process → zip_router_check_backup()
    ├─ Check if idle now?
    │  └─ Still busy (node queue has items)
    ├─ Reset timer for another 5 minutes
    └─ Keep waiting
    
[Eventually idle]
  → Execute backup
```

---

### **19. serial_line_event_message** - Keyboard Input (Debug)
**Khi nào**: User input từ terminal (chỉ khi compiled without __ASIX_C51__)  
**Làm gì**: Xử lý debug commands

**Các lệnh**:
```c
'l' - Send Router Advertisement
'x' - Exit gateway
'd' - Full network discovery
'f <node>' - Mark node as failing
'r' - Reset gateway
'b' - Print bridge status
'w' - Print provisioning list
's' - Smart start init
'e' - Get RF region
't' - Set RF region
'g' - Get TX power level
<default> - Print network info (IPs, neighbors, routes)
```

**Ví dụ**:
```
[User types 'd']
  → serial_line_event_message posted with data="d"
  → zip_process:
    └─ rd_full_network_discovery()
       ├─ Mark all nodes as STATUS_PROBE_PENDING
       ├─ Start probing nodes one by one
       └─ Will trigger ZIP_EVENT_ALL_NODES_PROBED when done

[User types 'r']
  → Reset gateway (same as ZIP_EVENT_RESET)

[User types <Enter>]
  → Print debug info:
    - All IPv6 addresses
    - IPv4 address
    - Neighbor cache (IPv6 → MAC)
    - Routing table
    - Default routes
```

---

## Event Flow Diagrams

### **Startup Sequence**
```
[Gateway Starts]
    ↓
PROCESS_EVENT_INIT
    ↓
ZIP_Router_Reset()
    ├─ Load config
    ├─ Init Z-Wave
    ├─ Setup IPv6
    └─ Start DTLS/mDNS
    ↓
tcpip_event (TCP_READY)
    ↓
system_net_hook(1)  → Network interfaces up
    ↓
ClassicZIPNode_init()
bridge_init()
    ↓
ZIP_EVENT_BRIDGE_INITIALIZED
    ↓
ApplicationInitProtocols()  → Start Z/IP command handlers
    ↓
[Discovery Phase]
probe nodes 1..N
    ↓
ZIP_EVENT_ALL_NODES_PROBED
    ↓
DHCP for all nodes
    ↓
ZIP_EVENT_ALL_IPV4_ASSIGNED
    ↓
send_nodelist()  → Portal updated
    ↓
ZIP_EVENT_NETWORK_MANAGEMENT_DONE
    ↓
[Gateway Fully Operational]
```

### **Node Inclusion Flow**
```
[Controller: Add Node]
    ↓
NM_WAIT_FOR_ADD → NM_WAIT_FOR_PROTOCOL
    ↓
[New node included]
    ↓
Interview node → Get NIF, version, S2 keys
    ↓
ZIP_EVENT_NODE_PROBED (new node)
    ↓
Assign IPv4 via DHCP
    ↓
ZIP_EVENT_NODE_IPV4_ASSIGNED (new node)
    ↓
ZIP_EVENT_NETWORK_MANAGEMENT_DONE
    ↓
send_nodelist()  → Portal knows new node
```

### **Backup Request Flow**
```
[SIGUSR1 signal]
    ↓
ZIP_EVENT_BACKUP_REQUEST
    ↓
Mark ZGW_BU active
    ↓
zip_router_check_backup()
    ├─ Node queue busy? → Wait
    ├─ Mailbox busy? → Wait
    ├─ NM busy? → Wait
    └─ PowerLevel busy? → Wait
    ↓
[Each component finishes]
    ↓
ZIP_EVENT_COMPONENT_DONE
    ↓
zip_router_check_backup() again
    ├─ All idle? → YES
    └─ Execute backup NOW
    ↓
backup_and_overwrite_nvm()
    ↓
Backup complete
```

---

## Component State Tracking

`zip_process` sử dụng `zgw_component_*` functions để track state:

```c
typedef enum {
  ZGW_BOOTING,           // Initial boot
  ZGW_ROUTER_RESET,      // Gateway reset
  ZGW_BRIDGE,            // Bridge initialization
  ZGW_NM,                // Network Management
  ZGW_BU,                // Backup
  ZGW_SHUTTING_DOWN      // Shutdown
} zgw_component_t;

// API:
zgw_component_start(component);   // Mark component active
zgw_component_done(component);    // Mark component done
ZGW_COMPONENT_ACTIVE(component);  // Check if active
```

**Ví dụ sử dụng**:
```c
// Start reset
zgw_component_start(ZGW_ROUTER_RESET);
ZIP_Router_Reset();

// ... reset completes ...

// Mark done
zgw_component_done(ZGW_ROUTER_RESET, NULL);
```

---

## Tổng kết

### **Vai trò của zip_process**:
1. **Event Dispatcher**: Nhận và xử lý 21 loại events
2. **State Coordinator**: Quản lý lifecycle của gateway
3. **Component Orchestrator**: Điều phối hoạt động của các subsystems
4. **Network Manager**: Xử lý network changes và updates
5. **Backup Controller**: Đảm bảo backup an toàn khi idle

### **Đặc điểm kỹ thuật**:
- **Event-driven**: Không busy-wait, chỉ wake khi có event
- **Asynchronous**: Các operations không block main loop
- **Coordinated**: Đảm bảo các components không conflict
- **Persistent**: Auto-save state to NVM when needed
- **Resilient**: Xử lý errors và retry operations

### **Performance**:
- Lightweight: Chỉ consume CPU khi có event
- Scalable: Handle multiple simultaneous events via queue
- Responsive: Typical event handling < 1ms

---

## Files Liên Quan

| File | Chức năng |
|------|-----------|
| `ZIP_Router.c:1342` | Main zip_process implementation |
| `router_events.h` | Event definitions |
| `ZIP_Router.h` | Public interfaces |
| `CC_NetworkManagement.c` | Posts NM events |
| `ResourceDirectory.c` | Posts probe events |
| `node_queue.c` | Posts queue events |
| `Bridge.c` | Posts bridge events |
| `dhcpc2.c` | Posts DHCP events |
| `ZW_tcp_client.c` | Posts tunnel events |

