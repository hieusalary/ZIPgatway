# ZIP_PROCESS - Trung Tâm Điều Phối ZIP Gateway

## Tổng quan

`zip_process` là **MAIN CONTROL PROCESS** của ZIP Gateway - đóng vai trò như một **Event-Driven State Machine** quản lý toàn bộ lifecycle và hoạt động của gateway.

**Location**: `src/ZIP_Router.c:1342`
**Type**: Contiki PROCESS (event-driven thread)
**Auto-start**: Được start tự động qua `AUTOSTART_PROCESSES(&zip_process)`

---

## Quick Reference: Event Trigger Sources

| Event | Nguồn gọi | Context |
|-------|-----------|---------|
| **ZIP_EVENT_RESET** | CC_NetworkManagement.c:1262<br>CC_NetworkManagement.c:1302<br>CC_NetworkManagement.c:1311<br>ZW_ZIPApplication.c:372<br>ZIP_Router.c:1455 | Set Default complete<br>Learn mode complete<br>Learn timeout<br>SUC ID changed<br>User keyboard 'r' |
| **ZIP_EVENT_TUNNEL_READY** | ZW_tcp_client.c:276<br>CC_Portal.c:34<br>jni-glue.c:137 | SSL handshake success<br>Portal re-enabled<br>Android JNI ready |
| **ZIP_EVENT_BRIDGE_INITIALIZED** | Bridge.c:159<br>Bridge_temp_assoc.c:289 | Virtual nodes created<br>Temp assoc done |
| **ZIP_EVENT_QUEUE_UPDATED** | node_queue.c:198<br>node_queue.c:442<br>Mailbox.c:931<br>Mailbox.c:1141 | Command added to queue<br>Send complete<br>Mailbox wakeup<br>Mailbox has more msgs |
| **ZIP_EVENT_ALL_NODES_PROBED** | ResourceDirectory.c:1472<br>ResourceDirectory.c:1511 | All nodes interviewed<br>SUC ID assignment done |
| **ZIP_EVENT_NODE_PROBED** | ResourceDirectory.c:2026<br>CC_NetworkManagement.c:815 | Single node probe done<br>NM node interview done |
| **ZIP_EVENT_ALL_IPV4_ASSIGNED** | dhcpc2.c:498 | DHCP round complete |
| **ZIP_EVENT_NODE_IPV4_ASSIGNED** | dhcpc2.c (multiple) | Single node got IPv4 |
| **ZIP_EVENT_NETWORK_MANAGEMENT_DONE** | CC_NetworkManagement.c:1800 | NM state → NM_IDLE |
| **ZIP_EVENT_NEW_NETWORK** | CC_NetworkManagement.c:1259 | Learn into new network |
| **ZIP_EVENT_COMPONENT_DONE** | node_queue.c:515<br>CC_NetworkManagement.c:3383 | Component finished task |
| **ZIP_EVENT_BACKUP_REQUEST** | Signal handler (SIGUSR1) | Admin requests backup |

---

## Kiến trúc Event-Driven

```c
PROCESS_THREAD(zip_process, ev, data)
{
  PROCESS_BEGIN();
  
  while(1) {
    // Process các events
    if (ev == ZIP_EVENT_RESET) { ... }
    else if (ev == ZIP_EVENT_TUNNEL_READY) { ... }
    else if (ev == ZIP_EVENT_QUEUE_UPDATED) { ... }
    // ... 18+ events khác
    
    PROCESS_WAIT_EVENT();  // Block cho đến khi có event mới
  }
  
  PROCESS_END();
}
```

**Cơ chế hoạt động**:
1. **process_post()** - Component gọi để gửi event (async)
2. Event được queue trong Contiki event queue
3. **PROCESS_WAIT_EVENT()** - zip_process wake up khi có event
4. Event handler xử lý theo type
5. Quay lại PROCESS_WAIT_EVENT() để đợi event tiếp theo

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

#### **Khi nào được gọi** (5 nguồn):

**A) User nhấn 'r' keyboard**
```c
// ZIP_Router.c:1455 - keyboard handler
case 'r':
  process_post(&zip_process, ZIP_EVENT_RESET, 0);
```

**B) Set Default Complete** (Factory Reset)
```c
// CC_NetworkManagement.c:1262
case NM_SET_DEFAULT:
  if (ev == NM_EV_MDNS_EXIT) {
    ApplicationDefaultSet();      // Xóa toàn bộ NVM
    bridge_reset();               // Reset virtual nodes
    process_post_synch(&zip_process, ZIP_EVENT_NEW_NETWORK, 0);
    process_post(&zip_process, ZIP_EVENT_RESET, 0);  // ← GỌI Ở ĐÂY
    controller_role = SUC;
  }
```
**Context**: Sau khi mDNS exited, đã xóa hết NVM, cần restart toàn bộ gateway

**C) Learn Mode Complete (Gateway learned into new network)**
```c
// CC_NetworkManagement.c:1302
case NM_WAIT_FOR_SECURE_LEARN:
  if (ev == NM_EV_SECURITY_DONE) {
    if (new_network) {
      // Đã vào network mới
      process_post(&zip_process, ZIP_EVENT_RESET, 0);  // ← GỌI Ở ĐÂY
      SendReplyWhenNetworkIsUpdated();
    }
  }
```
**Context**: Gateway vừa learn vào network mới, cần reload configuration

**D) Learn Mode Timeout (SIS không probe gateway)**
```c
// CC_NetworkManagement.c:1311
case NM_WAIT_FOR_PROBE_BY_SIS:
  if (ev == NM_EV_TIMEOUT) {
    process_post(&zip_process, ZIP_EVENT_RESET, 0);  // ← GỌI Ở ĐÂY
    nms.state = NM_WAIT_FOR_OUR_PROBE;
  }
```
**Context**: Đợi SIS probe 6 seconds nhưng timeout, tự reset để probe bản thân

**E) SUC ID Changed (Gateway được set làm SUC)**
```c
// ZW_ZIPApplication.c:372
void ApplicationControllerUpdate(uint8_t bStatus, nodeid_t bNodeID) {
  case UPDATE_STATE_SUC_ID:
    if (bNodeID != 0 && bNodeID == MyNodeID) {
      // Gateway vừa được promote thành SUC
      process_post(&zip_process, ZIP_EVENT_RESET, 0);  // ← GỌI Ở ĐÂY
    }
}
```
**Context**: Controller update gateway thành SUC, cần reset để activate vai trò mới

---

#### **Event Handler xử lý như thế nào**:
```c
// ZIP_Router.c:1587
if (ev == ZIP_EVENT_RESET) {
  if (data == (void*) 0) {
    LOG_PRINTF("Resetting....\n");
    zgw_component_start(ZGW_ROUTER_RESET);    // Mark reset active
    
    if (!ZIP_Router_Reset()) {
      ERR_PRINTF("Fatal error\n");
      process_post(&zip_process, PROCESS_EVENT_EXIT, 0);  // Fatal → exit
    }
  }
}
```

**ZIP_Router_Reset() thực hiện**:
1. **Stop all services**: mDNS, DTLS, TCP client
2. **Re-read config**: `/usr/local/etc/zipgateway.cfg`
3. **Reinit Z-Wave**: Reload HomeID, NodeID, node list
4. **Rebuild routing**: Clear và rebuild routing tables
5. **Restart discovery**: Resource Directory probe lại tất cả nodes
6. **Restore network**: Bring up TAP, br-lan interfaces
7. **Restart protocols**: DHCP, IPv6 autoconfiguration

#### **Concrete Example Flow**:
```
[Scenario: Gateway learned into new network]

1. Controller sends LEARN_MODE_SET
   → Gateway enters learn mode
   
2. Primary controller includes gateway
   → New HomeID: 0x11223344
   → New NodeID: 5
   
3. Security bootstrapping completes
   → S2 keys exchanged
   
4. NM state machine → NM_WAIT_FOR_SECURE_LEARN
   → ev = NM_EV_SECURITY_DONE
   → new_network = TRUE
   
5. CC_NetworkManagement.c:1302
   → process_post(&zip_process, ZIP_EVENT_RESET, 0)
   
6. zip_process receives ZIP_EVENT_RESET
   → Call ZIP_Router_Reset()
     ├─ Stop mDNS (old network data invalid)
     ├─ Re-read config file
     ├─ Load new HomeID 0x11223344
     ├─ Rebuild node list from new controller
     ├─ Generate new ULA prefix for PAN
     ├─ Restart TAP interface
     └─ Start discovery
     
7. Gateway operational in new network
   → Can now communicate with new controller
   → Old network nodes unreachable
```

---

### **3. ZIP_EVENT_TUNNEL_READY** - Portal Tunnel Established

#### **Khi nào được gọi** (3 nguồn):

**A) TCP Client SSL Handshake Success**
```c
// ZW_tcp_client.c:276
void tcp_client_callback(...) {
  if (mbedtls_ssl_handshake(&ssl_client_context) == 0) {
    // SSL handshake thành công!
    stop_watchdog_timer();
    gisZIPRReady = ZIPR_READY;
    
    // Queue Gateway Config Status message
    if (uip_packetqueue_alloc(&ipv6TxQueue, 
                              (uint8_t*)&GwConfigStatus,
                              sizeof(GwConfigStatus), 
                              QUEUE_PACKET_TIMEOUT)) {
      process_post(&zip_process, ZIP_EVENT_TUNNEL_READY, 0);  // ← GỌI Ở ĐÂY
      LOG_PRINTF("Send the ZIPR Ready message to portal.\r\n");
    }
  }
}
```
**Context**: 
- TCP connection đến portal server established
- SSL/TLS handshake completed (mbedtls)
- Certificate validation passed
- Encrypted tunnel ready
- Gateway Config Status queued để gửi đến portal

**B) Portal Connection Restored (Reconnect)**
```c
// CC_Portal.c:34
void portal_enable() {
  if (tcp_client_is_connected()) {
    // Đã connected nhưng cần re-enable routing
    process_post(&zip_process, ZIP_EVENT_TUNNEL_READY, 0);  // ← GỌI Ở ĐÂY
  } else {
    tcp_client_connect();  // Start connection
  }
}
```
**Context**: Portal bị disable rồi được re-enable, tunnel vẫn còn active

**C) JNI Android/Java Integration (Mobile app)**
```c
// jni-glue.c:137
JNIEXPORT void JNICALL Java_..._setPortalReady(JNIEnv* env, jobject obj) {
  process_post(&zip_process, ZIP_EVENT_TUNNEL_READY, 0);  // ← GỌI Ở ĐÂY
}
```
**Context**: Android app báo portal tunnel đã sẵn sàng (mobile gateway)

---

#### **Event Handler xử lý như thế nào**:
```c
// ZIP_Router.c:1604
if (ev == ZIP_EVENT_TUNNEL_READY) {
  LOG_PRINTF("[HIEU] TUNNEL READY!\n");

  // 1. Refresh LAN/PAN addresses
  ZIP_eeprom_init();

  // 2. Reinit IPv6 addresses
  refresh_ipv6_addresses();

  // 3. Bring network interface up
  system_net_hook(1);

  // 4. Enable Network Management to send pending replies
  NetworkManagement_Init();
}
```

**Chi tiết các bước**:

**1. ZIP_eeprom_init()**
- Đọc lại từ EEPROM: used LAN addresses, used PAN addresses
- Cần làm vì có thể portal đã cấp IP mới

**2. refresh_ipv6_addresses()**
```c
void refresh_ipv6_addresses() {
  // Update global IPv6 address (from portal)
  uip_ds6_addr_add(&cfg.lan_addr, 0, ADDR_MANUAL);
  
  // Update ULA prefix for Z-Wave PAN
  uip_ds6_prefix_add(&cfg.pan_prefix, 64, 0);
  
  // Update router advertisement daemon
  radvd_update_config();
}
```

**3. system_net_hook(1)**
```c
// Executes: /usr/local/etc/zipgateway.tun
// Script brings up network:
//   - Add routes to portal
//   - Setup NAT rules
//   - Configure firewall
//   - Enable IPv6 forwarding
```

**4. NetworkManagement_Init()**
- Cho phép gửi các NM replies đang pending
- Trước đó blocked vì chưa có tunnel

#### **Concrete Example Flow**:
```
[Scenario: Gateway connects to portal at startup]

1. Gateway boots up
   → Reads config: portal_url = "zipr.z-wave.me:44123"
   
2. TCP client starts connection
   → Resolve DNS: zipr.z-wave.me → 52.58.19.123
   → TCP SYN → SYN-ACK → ACK (3-way handshake)
   → TCP connection established
   
3. SSL/TLS handshake begins
   → ClientHello (cipher suites, TLS version)
   → ServerHello (selected cipher, certificate)
   → Certificate validation
     ├─ Check portal cert signature
     ├─ Check expiration date
     └─ Verify CA chain
   → Key exchange (ECDHE)
   → ChangeCipherSpec
   → Encrypted Finished messages
   
4. SSL handshake complete!
   → ZW_tcp_client.c:276
   → gisZIPRReady = ZIPR_READY
   → Queue GwConfigStatus packet (gateway info for portal)
   → process_post(&zip_process, ZIP_EVENT_TUNNEL_READY, 0)
   
5. zip_process receives ZIP_EVENT_TUNNEL_READY
   → ZIP_eeprom_init()
     └─ Load allocated LAN/PAN IPs from NVM
   
   → refresh_ipv6_addresses()
     ├─ Add LAN addr: 2001:db8:1::1/64 (from portal)
     ├─ Add PAN prefix: fd00:aaaa::/64
     └─ Update radvd.conf
     
   → system_net_hook(1)
     ├─ Execute /usr/local/etc/zipgateway.tun
     ├─ Script output:
     │   ifconfig tap0 up
     │   ip -6 route add default via fe80::1 dev tap0
     │   ip6tables -A FORWARD -i tap0 -j ACCEPT
     │   sysctl -w net.ipv6.conf.all.forwarding=1
     └─ Network interfaces operational
     
   → NetworkManagement_Init()
     └─ Send pending NODE_ADD reply to controller
     
6. Portal receives GwConfigStatus
   → Portal knows gateway is online
   → Portal can route packets to gateway's Z-Wave network
   → Controller can send commands via portal
   
[Result]: 
- Encrypted tunnel active
- IPv6 routing enabled
- Gateway reachable via portal
- Z-Wave nodes accessible remotely
```

#### **Timing Diagram**:
```
Gateway                 TCP Stack              Portal Server
  |                         |                        |
  |--TCP connect----------->|                        |
  |                         |--SYN------------------->|
  |                         |<--SYN-ACK---------------|
  |                         |--ACK------------------->|
  |                         |                        |
  |--SSL handshake--------->|<--Certificate----------|
  |                         |--ClientKeyExchange---->|
  |                         |<--ChangeCipherSpec-----|
  |                         |                        |
  |<-Handshake OK-----------|                        |
  |                         |                        |
  | process_post(ZIP_EVENT_TUNNEL_READY)            |
  |                         |                        |
  | ZIP_eeprom_init()       |                        |
  | refresh_ipv6_addresses()|                        |
  | system_net_hook(1)      |                        |
  | NetworkManagement_Init()|                        |
  |                         |                        |
  |--GwConfigStatus-------->|----------------------->|
  |                         |                        | Portal stores
  |                         |                        | gateway info
  |<---------------------------ACK-------------------|
  |                         |                        |
  [Tunnel operational - can forward packets]
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

#### **Khi nào được gọi** (4 nguồn):

**A) New Command Added to Queue**
```c
// node_queue.c:198
bool node_queue_add(nodeid_t node, uint8_t* buf, uint16_t len) {
  // Add packet to queue
  if (use_long_q) {
    rc = uip_packetqueue_alloc(&long_queue, 
                                &uip_buf[UIP_LLH_LEN], 
                                uip_len, 
                                60000);  // 60 second timeout
  } else {
    rc = uip_packetqueue_alloc(&first_attempt_queue, 
                                &uip_buf[UIP_LLH_LEN], 
                                uip_len, 
                                60000);
  }
  
  // Notify zip_process to start sending
  process_post(&zip_process, ZIP_EVENT_QUEUE_UPDATED, 0);  // ← GỌI Ở ĐÂY
  return rc;
}
```
**Context**: 
- Controller gửi command đến node (ví dụ: BASIC_SET, SWITCH_BINARY_SET)
- Command được queue vào `first_attempt_queue` hoặc `long_queue`
- Cần trigger zip_process để xử lý queue

**Ví dụ**:
```
Controller sends: BASIC_SET(value=0xFF) to Node 5
  → ClassicZIPNode_CallBack() receives packet
  → node_queue_add(node=5, cmd=BASIC_SET)
  → Packet queued in first_attempt_queue
  → process_post(ZIP_EVENT_QUEUE_UPDATED)
```

---

**B) Command Send Complete (Success hoặc Fail)**
```c
// node_queue.c:442
static void tx_done_callback(uint8_t status, void* user, TX_STATUS_TYPE *t) {
  if (queue_state == QS_SENDING_FIRST) {
    // Command from first_attempt_queue sent
    if (status == TRANSMIT_COMPLETE_OK) {
      uip_packetqueue_pop(&first_attempt_queue);  // Remove successfully sent
    } else {
      // Failed, move to long_queue for retry
      move_to_long_queue();
    }
  } else if (queue_state == QS_SENDING_LONG) {
    // Command from long_queue sent (no matter result, remove it)
    uip_packetqueue_pop(&long_queue);
  }
  
  queue_state = QS_IDLE;
  
  // Process next command in queue
  process_post(&zip_process, ZIP_EVENT_QUEUE_UPDATED, 0);  // ← GỌI Ở ĐÂY
}
```
**Context**:
- Z-Wave RF transmission complete (ACK hoặc NACK)
- Pop packet khỏi queue
- Cần process next command

**Ví dụ**:
```
[Command sending sequence]
Queue: [BASIC_SET→Node5] [SWITCH_SET→Node7] [VERSION_GET→Node3]

1. Send BASIC_SET → Node 5
   → RF transmission → ACK received
   → tx_done_callback(TRANSMIT_COMPLETE_OK)
   → Pop from queue
   → process_post(ZIP_EVENT_QUEUE_UPDATED)  ← Trigger next
   
2. Send SWITCH_SET → Node 7
   → RF transmission → NACK (node sleeping)
   → tx_done_callback(TRANSMIT_COMPLETE_NO_ACK)
   → Move to long_queue (retry later)
   → process_post(ZIP_EVENT_QUEUE_UPDATED)  ← Trigger next
   
3. Send VERSION_GET → Node 3
   ...
```

---

**C) Mailbox Has Messages to Send**
```c
// Mailbox.c:931
static void mb_state_machine(mb_event_t event) {
  switch (mb_state.state) {
    case MB_STATE_IDLE:
      if (event == MB_EVENT_TRANSITION) {
        // Mailbox cần gửi message
        process_post(&zip_process, ZIP_EVENT_QUEUE_UPDATED, 0);  // ← GỌI Ở ĐÂY
      }
      break;
  }
}
```
**Context**:
- Mailbox service nhận được wake-up notification từ sleeping node
- Cần gửi queued messages đến node
- Trigger queue processing

**Ví dụ**:
```
[Sleeping node scenario]
1. Controller sends command to Node 10 (battery device, sleeping)
   → Command queued in mailbox (node not awake)
   
2. Later: Node 10 wakes up
   → Sends WAKE_UP_NOTIFICATION to gateway
   
3. Gateway receives notification
   → mb_state_machine(MB_EVENT_WAKEUP_RECEIVED)
   → mb_state.state = MB_STATE_IDLE
   → event = MB_EVENT_TRANSITION
   → process_post(ZIP_EVENT_QUEUE_UPDATED)  ← Tell zip_process
   
4. zip_process handles event
   → process_node_queues()
   → Check mailbox for Node 10
   → Send queued command(s) to Node 10
   → Node 10 receives and goes back to sleep
```

---

**D) Mailbox Service Ready to Send**
```c
// Mailbox.c:1141
void mb_push_event(mb_event_t event, uint8_t *data, uint16_t len) {
  if (event == MB_EVENT_SEND_DONE) {
    // Previous send done, check if more messages
    if (mb_has_more_messages()) {
      process_post(&zip_process, ZIP_EVENT_QUEUE_UPDATED, 0);  // ← GỌI Ở ĐÂY
    }
  }
}
```

---

#### **Event Handler xử lý như thế nào**:
```c
// ZIP_Router.c:1682
if (ev == ZIP_EVENT_QUEUE_UPDATED) {
  process_node_queues();  // ← Main queue processing function
}
```

**process_node_queues() thực hiện**:
```c
void process_node_queues() {
  if (queue_state != QS_IDLE) {
    return;  // Already sending, don't interrupt
  }
  
  // Priority 1: Check mailbox
  if (mb_enabled() && mb_wakeup_event_pending()) {
    mb_send_wakeup_event();
    return;
  }
  
  // Priority 2: Send from first_attempt_queue
  if (!uip_packetqueue_is_empty(&first_attempt_queue)) {
    uint8_t *data = uip_packetqueue_data(&first_attempt_queue);
    uint16_t len = uip_packetqueue_datalen(&first_attempt_queue);
    
    queue_state = QS_SENDING_FIRST;
    
    // Send via Z-Wave RF
    if (!ClassicZIPNode_sendRequest(data, len, tx_done_callback)) {
      // Send failed immediately
      tx_done_callback(TRANSMIT_COMPLETE_FAIL, NULL, NULL);
    }
    return;
  }
  
  // Priority 3: Send from long_queue (retries)
  if (!uip_packetqueue_is_empty(&long_queue)) {
    uint8_t *data = uip_packetqueue_data(&long_queue);
    uint16_t len = uip_packetqueue_datalen(&long_queue);
    
    queue_state = QS_SENDING_LONG;
    
    if (!ClassicZIPNode_sendRequest(data, len, tx_done_callback)) {
      tx_done_callback(TRANSMIT_COMPLETE_FAIL, NULL, NULL);
    }
    return;
  }
  
  // All queues empty
  queue_state = QS_IDLE;
}
```

#### **Queue Priority**:
1. **Mailbox** (highest) - Sleeping nodes that just woke up
2. **first_attempt_queue** - New commands, first try
3. **long_queue** (lowest) - Failed commands, retries

#### **Concrete Example Flow**:
```
[Scenario: Multiple commands queued]

Initial state:
  first_attempt_queue: [BASIC_SET→Node5] [SWITCH_SET→Node7]
  long_queue: [VERSION_GET→Node3] (retry from earlier failure)
  mailbox: Node 10 just woke up, has [SENSOR_GET] waiting

---

Event 1: Node 10 wakes up
  → Mailbox.c detects wake-up
  → process_post(&zip_process, ZIP_EVENT_QUEUE_UPDATED, 0)
  
  zip_process → process_node_queues()
    ├─ Check mailbox: YES, Node 10 awake
    ├─ Priority 1: Send [SENSOR_GET] to Node 10
    └─ queue_state = QS_SENDING_FIRST

---

Event 2: SENSOR_GET send complete (ACK)
  → tx_done_callback(TRANSMIT_COMPLETE_OK)
  → process_post(&zip_process, ZIP_EVENT_QUEUE_UPDATED, 0)
  
  zip_process → process_node_queues()
    ├─ Check mailbox: Empty now
    ├─ Priority 2: first_attempt_queue
    ├─ Pop [BASIC_SET→Node5]
    └─ Send BASIC_SET to Node 5

---

Event 3: BASIC_SET send complete (ACK)
  → tx_done_callback(TRANSMIT_COMPLETE_OK)
  → process_post(&zip_process, ZIP_EVENT_QUEUE_UPDATED, 0)
  
  zip_process → process_node_queues()
    ├─ Check first_attempt_queue
    ├─ Pop [SWITCH_SET→Node7]
    └─ Send SWITCH_SET to Node 7

---

Event 4: SWITCH_SET send FAILED (NACK - Node 7 sleeping)
  → tx_done_callback(TRANSMIT_COMPLETE_NO_ACK)
  → Move to long_queue
  → process_post(&zip_process, ZIP_EVENT_QUEUE_UPDATED, 0)
  
  zip_process → process_node_queues()
    ├─ first_attempt_queue: Empty
    ├─ Priority 3: long_queue
    │   Now has: [VERSION_GET→Node3] [SWITCH_SET→Node7]
    ├─ Pop [VERSION_GET→Node3]
    └─ Send VERSION_GET to Node 3

---

Event 5: VERSION_GET send complete (ACK)
  → process_post(&zip_process, ZIP_EVENT_QUEUE_UPDATED, 0)
  
  zip_process → process_node_queues()
    ├─ long_queue: [SWITCH_SET→Node7]
    ├─ Pop [SWITCH_SET→Node7]
    └─ Send SWITCH_SET to Node 7 (retry)

---

Event 6: SWITCH_SET still FAILED (Node 7 still sleeping)
  → Remove from long_queue (max retries reached)
  → process_post(&zip_process, ZIP_EVENT_QUEUE_UPDATED, 0)
  
  zip_process → process_node_queues()
    ├─ All queues empty
    └─ queue_state = QS_IDLE

---

Final state: All commands processed
```

#### **Timeout Handling**:
```c
// Commands in queue have 60 second timeout
uip_packetqueue_alloc(&queue, buf, len, 60000);  // 60000ms

// If timeout expires before send:
void queue_timeout_callback() {
  uip_packetqueue_pop(&queue);  // Remove expired
  process_post(&zip_process, ZIP_EVENT_QUEUE_UPDATED, 0);  // Process next
}
```

---

### **7. ZIP_EVENT_ALL_NODES_PROBED** - Discovery Complete

#### **Khi nào được gọi** (2 nguồn):

**A) Normal Discovery Complete**
```c
// ResourceDirectory.c:1472
static void rd_probe_next_node() {
  rd_node_database_entry_t* nd;
  
  // Loop through all nodes
  for (i = 0; i < ZW_MAX_NODES; i++) {
    nd = &nlist[i];
    
    if (nd->state == STATUS_PROBE_PENDING) {
      // Found node to probe, start probing
      rd_node_probe_update(nd);
      goto exit;  // Exit without posting event
    }
  }
  
  // No more nodes to probe - DISCOVERY COMPLETE!
  process_post(&zip_process, ZIP_EVENT_ALL_NODES_PROBED, 0);  // ← GỌI Ở ĐÂY
  
exit:
  return;
}
```
**Context**:
- Gateway startup hoặc full discovery triggered
- Resource Directory probe từng node một
- Khi không còn node nào STATUS_PROBE_PENDING → Discovery xong

**Probe process cho mỗi node**:
```c
void rd_node_probe_update(rd_node_database_entry_t* nd) {
  // Step 1: Get Node Information Frame
  ZW_RequestNodeInfo(nd->nodeid);
  
  // Step 2: Get Version Info
  send_version_get(nd->nodeid);
  
  // Step 3: Get Security S2 keys (if supported)
  send_s2_key_get(nd->nodeid);
  
  // Step 4: Get endpoints (Multi Channel)
  send_multi_channel_get(nd->nodeid);
  
  // Step 5: Get command classes for each endpoint
  for (each endpoint) {
    send_nif_get(endpoint);
  }
  
  // When all done:
  nd->state = STATUS_DONE;
  process_post(&zip_process, ZIP_EVENT_NODE_PROBED, nd);
}
```

---

**B) Forced Discovery Complete (With SUC/SIS update)**
```c
// ResourceDirectory.c:1511
static void send_suc_id_complete() {
  if (all_nodes_sent_suc_id) {
    // Đã gửi SUC ID cho tất cả nodes
    // hoặc gán SUC return routes
    
    process_post(&zip_process, ZIP_EVENT_ALL_NODES_PROBED, 0);  // ← GỌI Ở ĐÂY
  }
}
```
**Context**:
- Gateway là SUC (Static Update Controller)
- Cần inform tất cả nodes về SUC NodeID
- Hoặc assign SUC return routes

---

#### **Event Handler xử lý như thế nào**:
```c
// ZIP_Router.c:1726
if (ev == ZIP_EVENT_ALL_NODES_PROBED) {
  LOG_PRINTF("All nodes have been probed\n");
  
  // 1. Notify Network Management
  NetworkManagement_all_nodes_probed();
  
  // 2. Update network flags
  NetworkManagement_NetworkUpdateStatusUpdate(NETWORK_UPDATE_FLAG_PROBE);
  
  // 3. Enable SmartStart if was disabled
  NetworkManagement_smart_start_init_if_pending();
  
  // 4. Send node list to portal (if all nodes have IPv4)
  if (ipv46nat_all_nodes_has_ip()) {
    send_nodelist();
  }
  
  // 5. Restart node queue processing
  process_post(&zip_process, ZIP_EVENT_QUEUE_UPDATED, 0);
  
  // 6. Check if backup requested
  zip_router_check_backup(data);
}
```

**Chi tiết các bước**:

**1. NetworkManagement_all_nodes_probed()**
```c
void NetworkManagement_all_nodes_probed() {
  if (nms.state == NM_WAIT_FOR_PROBE) {
    // NM was waiting for probe to complete
    nms.state = NM_IDLE;
    send_reply();  // Send NODE_ADD reply to controller
  }
}
```

**2. NetworkManagement_NetworkUpdateStatusUpdate()**
```c
// Set PROBE flag in network status
// Portal can query this to know discovery is done
network_status_flags |= NETWORK_UPDATE_FLAG_PROBE;
```

**3. NetworkManagement_smart_start_init_if_pending()**
```c
// SmartStart was disabled during discovery
// Now re-enable it so new devices can join
ZW_AddNodeToNetwork(ADD_NODE_SMART_START, callback);
```

**4. send_nodelist()**
```c
void send_nodelist() {
  // Build UNSOLICITED_DESTINATION_LIST
  // Contains: HomeID, nodes, IPs, command classes
  
  for (each node) {
    add_node_info_to_list(node);
  }
  
  // Send to portal via tunnel
  send_to_portal(&nodelist, len);
}
```

**5. Restart queue**
- Commands may have been blocked during discovery
- Now process pending commands

**6. Backup check**
- Discovery complete = good time to backup
- Check if backup was requested

---

#### **Concrete Example Flow**:
```
[Scenario: Gateway boots up with 5 nodes in network]

Startup sequence:
  1. Gateway loads node list from NVM: Nodes 1,2,3,4,5
  2. Mark all nodes: STATUS_PROBE_PENDING
  3. Start discovery

---

Discovery Phase:

T=0s: Probe Node 1 (Gateway itself)
  → Send NIF request
  → Get version: SDK 7.18.01
  → State: STATUS_DONE
  → process_post(ZIP_EVENT_NODE_PROBED, node1)
  → rd_probe_next_node() called

T=2s: Probe Node 2 (Door Lock)
  → Send NIF request → Response: [DOOR_LOCK, BATTERY, USER_CODE]
  → Send VERSION_GET → Response: v4
  → Send SECURITY_2_KEY_GET → Response: [S2_AUTHENTICATED, S2_ACCESS_CONTROL]
  → Send MULTI_CHANNEL_GET → Response: No endpoints
  → State: STATUS_DONE
  → process_post(ZIP_EVENT_NODE_PROBED, node2)
  → rd_probe_next_node() called

T=5s: Probe Node 3 (Motion Sensor - Battery)
  → Send NIF request → Sleeping, FAILED
  → State: STATUS_PROBE_FAIL → Will retry later
  → rd_probe_next_node() called

T=7s: Probe Node 4 (Light Switch)
  → Send NIF request → Response: [SWITCH_BINARY, SWITCH_MULTILEVEL]
  → Send VERSION_GET → Response: v5
  → Send S2_KEY_GET → Response: [S2_AUTHENTICATED]
  → Send MULTI_CHANNEL_GET → Response: 2 endpoints
  → For each endpoint: Get NIF
  → State: STATUS_DONE
  → process_post(ZIP_EVENT_NODE_PROBED, node4)
  → rd_probe_next_node() called

T=12s: Probe Node 5 (Thermostat)
  → Send NIF request → Response: [THERMOSTAT_MODE, SENSOR_MULTILEVEL]
  → Send VERSION_GET → Response: v6
  → Send S2_KEY_GET → Response: [S2_AUTHENTICATED]
  → State: STATUS_DONE
  → process_post(ZIP_EVENT_NODE_PROBED, node5)
  → rd_probe_next_node() called

T=14s: Node 3 still pending (sleeping)
  → Skip for now
  → NO MORE NODES to probe immediately
  
  → process_post(&zip_process, ZIP_EVENT_ALL_NODES_PROBED, 0)  ← EVENT FIRED

---

zip_process receives ZIP_EVENT_ALL_NODES_PROBED:

  → NetworkManagement_all_nodes_probed()
    └─ Update NM state machine
    
  → NetworkManagement_NetworkUpdateStatusUpdate(NETWORK_UPDATE_FLAG_PROBE)
    └─ Set flag: Probe phase done
    
  → NetworkManagement_smart_start_init_if_pending()
    └─ Re-enable SmartStart learn mode
    
  → Check: ipv46nat_all_nodes_has_ip()?
    ├─ Node 1: 192.168.1.100 ✓
    ├─ Node 2: 192.168.1.101 ✓
    ├─ Node 3: No IP (sleeping)
    ├─ Node 4: 192.168.1.102 ✓
    └─ Node 5: 192.168.1.103 ✓
    
    Result: NOT all have IP → Don't send nodelist yet
    
  → process_post(ZIP_EVENT_QUEUE_UPDATED)
    └─ Process pending commands
    
  → zip_router_check_backup()
    └─ No backup pending

---

Later: Node 3 wakes up

T=300s: Node 3 sends WAKE_UP_NOTIFICATION
  → Gateway probes Node 3
  → Probe completes: STATUS_DONE
  → process_post(ZIP_EVENT_NODE_PROBED, node3)
  
  → All nodes now probed!
  → Node 3 gets IPv4: 192.168.1.104
  → process_post(ZIP_EVENT_NODE_IPV4_ASSIGNED, node3)
  
  → Now ipv46nat_all_nodes_has_ip() = TRUE
  → send_nodelist() to portal
  
Portal receives node list:
{
  "homeID": "0xAABBCCDD",
  "nodes": [
    {"id": 1, "ip": "192.168.1.100", "classes": [...]},
    {"id": 2, "ip": "192.168.1.101", "classes": [DOOR_LOCK, ...]},
    {"id": 3, "ip": "192.168.1.104", "classes": [SENSOR_BINARY, ...]},
    {"id": 4, "ip": "192.168.1.102", "classes": [SWITCH_BINARY, ...]},
    {"id": 5, "ip": "192.168.1.103", "classes": [THERMOSTAT_MODE, ...]}
  ]
}

[Discovery Complete - Gateway Fully Operational]
```

#### **Discovery State Machine**:
```
For each node:

  STATUS_CREATED
      ↓
  STATUS_PROBE_PENDING ────┐
      ↓                    │
  [Send NIF request]       │
      ↓                    │
  [Wait response]          │
      ↓                    │
  Success? ───No──→ STATUS_PROBE_FAIL → Retry later ─┘
      ↓ Yes
  [Get Version]
      ↓
  [Get S2 Keys]
      ↓
  [Get Endpoints]
      ↓
  [Get Command Classes]
      ↓
  STATUS_DONE
      ↓
  process_post(ZIP_EVENT_NODE_PROBED)
      ↓
  Next node...

When all nodes probed or failed:
  → process_post(ZIP_EVENT_ALL_NODES_PROBED)
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

#### **Khi nào được gọi**:

**DHCP Round Complete**
```c
// dhcpc2.c:498
static void dhcpc_oneshot_done() {
  // DHCP client đã request IPv4 cho tất cả nodes
  // Có thể thành công, fail, hoặc timeout
  
  for (each node) {
    if (dhcp_request_sent && !dhcp_ack_received) {
      dhcpc_state.renew_failed++;  // Count failures
    }
  }
  
  // Update timers for next DHCP round
  if (dhcpc_state.renew_failed == 0) {
    // All successful
    dhcpc_state.ticks = dhcpc_state.lease_time;
    dhcpc_state.timeout = dhcpc_state.lease_time / 2;
  }
  
  // Notify zip_process
  process_post(&zip_process, ZIP_EVENT_ALL_IPV4_ASSIGNED, 0);  // ← GỌI Ở ĐÂY
}
```

**Context của DHCP flow**:
```c
// dhcpc2.c - DHCP State Machine

// State 1: DISCOVER
void dhcpc_discover() {
  for (each Z-Wave node) {
    if (!has_ipv4) {
      send_dhcp_discover(node);
    }
  }
}

// State 2: REQUEST
void dhcpc_request_received(offer) {
  send_dhcp_request(offered_ip);
}

// State 3: ACK
void dhcpc_ack_received(ack) {
  assign_ipv4_to_node(node, acked_ip);
  process_post(&zip_process, ZIP_EVENT_NODE_IPV4_ASSIGNED, node);
}

// State 4: COMPLETE (all nodes processed)
void dhcpc_oneshot_done() {
  process_post(&zip_process, ZIP_EVENT_ALL_IPV4_ASSIGNED, 0);  // ← HERE
}
```

**Timing**:
- Discovery: 0-2s per node
- Request: 0-1s per node
- Total: ~5-30s depending on number of nodes

---

#### **Event Handler xử lý như thế nào**:
```c
// ZIP_Router.c:1699
if (ev == ZIP_EVENT_ALL_IPV4_ASSIGNED) {
  LOG_PRINTF("ZIP_EVENT_ALL_IPV4_ASSIGNED\n");

  // 1. Execute external script with gateway's IPv4
  char ip[IPV6_STR_LEN];
  snprintf(ip, IPV6_STR_LEN, "IP=%d.%d.%d.%d", 
           uip_hostaddr.u8[0], uip_hostaddr.u8[1], 
           uip_hostaddr.u8[2], uip_hostaddr.u8[3]);
  
  char *args[] = {linux_conf_fin_script, NULL};
  char *env[] = {ip, NULL};
  
  if (fork() == 0) {
    // Child process
    execve(linux_conf_fin_script, args, env);  // Execute script
    exit(1);
  }
  
  // 2. Check pending IPv4 packets
  udp_server_check_ipv4_queue();
  
  // 3. Send node list if probe done
  if (!rd_probe_in_progress()) {
    send_nodelist();
  }
  
  // 4. Update network status
  NetworkManagement_NetworkUpdateStatusUpdate(NETWORK_UPDATE_FLAG_DHCPv4);
}
```

**Chi tiết các bước**:

**1. Execute zipgateway.fin script**
```bash
#!/bin/sh
# /usr/local/etc/zipgateway.fin
# Receives environment variable: IP=192.168.1.50

echo "Gateway IPv4: $IP"

# Update firewall rules
iptables -A FORWARD -s $IP -j ACCEPT

# Update DNS
echo "nameserver $IP" >> /etc/resolv.conf

# Notify external monitoring
curl -X POST http://monitor.local/gateway-ready -d "ip=$IP"
```

**2. udp_server_check_ipv4_queue()**
```c
void udp_server_check_ipv4_queue() {
  // Check for packets that were queued waiting for IPv4
  
  for (packet in ipv4_pending_queue) {
    if (destination_has_ipv4_now(packet.dest_node)) {
      // Now can send!
      send_packet(packet);
      remove_from_queue(packet);
    }
  }
}
```

**3. send_nodelist()**
- Gửi UNSOLICITED_DESTINATION_LIST đến portal
- Chứa IPv4 addresses của tất cả nodes
- Portal cập nhật routing table

**4. Update network status**
```c
network_status_flags |= NETWORK_UPDATE_FLAG_DHCPv4;
// Portal queries this flag to know DHCP is complete
```

---

#### **Concrete Example Flow**:
```
[Scenario: Gateway with 3 Z-Wave nodes needs IPv4]

Network topology:
  - Gateway: br-lan interface, needs DHCP
  - Node 2: Door Lock, needs DHCP
  - Node 3: Light Switch, needs DHCP
  - Node 4: Motion Sensor, needs DHCP
  
DHCP Server: 192.168.1.1 (router on LAN)

---

T=0s: DHCP Discovery Phase

Gateway sends DHCP DISCOVER for itself:
  DHCP DISCOVER
    chaddr: 02:00:00:00:00:01 (gateway MAC)
    options: [DHCP_DISCOVER, hostname="zipgateway"]

Gateway sends DHCP DISCOVER for Node 2:
  DHCP DISCOVER
    chaddr: 02:00:00:00:00:02 (virtual MAC for node 2)
    options: [DHCP_DISCOVER, hostname="zwave-node-2"]

Gateway sends DHCP DISCOVER for Node 3:
  DHCP DISCOVER
    chaddr: 02:00:00:00:00:03
    options: [DHCP_DISCOVER, hostname="zwave-node-3"]

Gateway sends DHCP DISCOVER for Node 4:
  DHCP DISCOVER
    chaddr: 02:00:00:00:00:04
    options: [DHCP_DISCOVER, hostname="zwave-node-4"]

---

T=1s: DHCP Server Responds with OFFERS

DHCP Server → Gateway:
  DHCP OFFER: 192.168.1.50 for chaddr:02:00:00:00:00:01
  DHCP OFFER: 192.168.1.101 for chaddr:02:00:00:00:00:02
  DHCP OFFER: 192.168.1.102 for chaddr:02:00:00:00:00:03
  DHCP OFFER: 192.168.1.103 for chaddr:02:00:00:00:00:04

---

T=2s: Gateway Sends DHCP REQUESTS

Gateway → DHCP Server:
  DHCP REQUEST: Accept 192.168.1.50 for gateway
  DHCP REQUEST: Accept 192.168.1.101 for node 2
  DHCP REQUEST: Accept 192.168.1.102 for node 3
  DHCP REQUEST: Accept 192.168.1.103 for node 4

---

T=3s: DHCP Server Sends ACKs

DHCP Server → Gateway:
  DHCP ACK: 192.168.1.50, lease_time=86400 (24 hours)
    → process_post(ZIP_EVENT_NODE_IPV4_ASSIGNED, gateway)
    → uip_hostaddr = 192.168.1.50
    
  DHCP ACK: 192.168.1.101, lease_time=86400
    → process_post(ZIP_EVENT_NODE_IPV4_ASSIGNED, node2)
    → Node 2 IPv4 mapping: fd00:aaaa::2 ↔ 192.168.1.101
    
  DHCP ACK: 192.168.1.102, lease_time=86400
    → process_post(ZIP_EVENT_NODE_IPV4_ASSIGNED, node3)
    → Node 3 IPv4 mapping: fd00:aaaa::3 ↔ 192.168.1.102
    
  DHCP ACK: 192.168.1.103, lease_time=86400
    → process_post(ZIP_EVENT_NODE_IPV4_ASSIGNED, node4)
    → Node 4 IPv4 mapping: fd00:aaaa::4 ↔ 192.168.1.103

---

T=5s: DHCP Round Complete

All nodes have IPv4 assigned!

dhcpc_oneshot_done() called:
  → dhcpc_state.renew_failed = 0  (no failures)
  → dhcpc_state.ticks = 86400  (24 hours)
  → dhcpc_state.timeout = 43200  (12 hours - renew at 50%)
  
  → process_post(&zip_process, ZIP_EVENT_ALL_IPV4_ASSIGNED, 0)

---

zip_process receives ZIP_EVENT_ALL_IPV4_ASSIGNED:

Step 1: Execute zipgateway.fin script
  → Fork child process
  → Child executes: /usr/local/etc/zipgateway.fin
  → Environment: IP=192.168.1.50
  
  Script output:
    Gateway IPv4: 192.168.1.50
    Adding firewall rule for 192.168.1.50
    Updating DNS configuration
    Notifying monitoring system
    
  → Child exits

Step 2: Check IPv4 pending queue
  → udp_server_check_ipv4_queue()
  
  Found pending packet:
    Dest: Node 2 (was queued because no IPv4 yet)
    Now Node 2 has 192.168.1.101
    → Convert: ::ffff:192.168.1.101
    → Send packet now!

Step 3: Send node list to portal
  → Check: rd_probe_in_progress()? NO
  → send_nodelist()
  
  Node list packet to portal:
  {
    "homeID": "0xAABBCCDD",
    "gateway_ipv4": "192.168.1.50",
    "nodes": [
      {
        "id": 2,
        "ipv6": "fd00:aaaa::2",
        "ipv4": "192.168.1.101",
        "classes": [DOOR_LOCK, BATTERY, USER_CODE]
      },
      {
        "id": 3,
        "ipv6": "fd00:aaaa::3",
        "ipv4": "192.168.1.102",
        "classes": [SWITCH_BINARY, SWITCH_MULTILEVEL]
      },
      {
        "id": 4,
        "ipv6": "fd00:aaaa::4",
        "ipv4": "192.168.1.103",
        "classes": [SENSOR_BINARY, BATTERY]
      }
    ]
  }

Step 4: Update network status
  → NetworkManagement_NetworkUpdateStatusUpdate(NETWORK_UPDATE_FLAG_DHCPv4)
  → network_status_flags |= NETWORK_UPDATE_FLAG_DHCPv4
  
  Portal can now query status and see:
    - Probe: ✓ Complete
    - DHCPv4: ✓ Complete
    - Bridge: ✓ Initialized

---

[DHCP Complete - Gateway and All Nodes Have IPv4]

Routing table now:
  fd00:aaaa::2 ↔ ::ffff:192.168.1.101 → Node 2
  fd00:aaaa::3 ↔ ::ffff:192.168.1.102 → Node 3
  fd00:aaaa::4 ↔ ::ffff:192.168.1.103 → Node 4

Controller can now use either:
  - IPv6: fd00:aaaa::2
  - IPv4-mapped: ::ffff:192.168.1.101
  
Both route to same node!
```

---

#### **DHCP Renewal**:
```c
// After 12 hours (50% of lease time)
void dhcpc_periodic() {
  if (dhcpc_state.ticks > dhcpc_state.timeout) {
    // Time to renew
    for (each node with IPv4) {
      send_dhcp_request(node, RENEWING);
    }
  }
  
  dhcpc_state.ticks++;
}

// After renewal completes
dhcpc_oneshot_done() {
  process_post(&zip_process, ZIP_EVENT_ALL_IPV4_ASSIGNED, 0);
  // Same event, but for renewal
}
```

#### **DHCP Failure Handling**:
```c
// If some nodes fail to get IPv4
dhcpc_oneshot_done() {
  if (dhcpc_state.renew_failed > 0) {
    // Some nodes didn't get IP
    // Don't reset lease timer, will retry sooner
    LOG_PRINTF("%d nodes failed DHCP\n", dhcpc_state.renew_failed);
  }
  
  // Still post event even with failures
  process_post(&zip_process, ZIP_EVENT_ALL_IPV4_ASSIGNED, 0);
}
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

