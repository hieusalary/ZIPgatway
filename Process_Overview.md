Tôi sẽ trace toàn bộ flow khởi động Contiki và xử lý gói tin, từ `main()` cho đến khi packet được deliver đến application. 

Created 6 todos

Read [](file:///c%3A/vscode_src/zipgateway-main/contiki/platform/linux/contiki-main.c#100-100), lines 100 to 145

Read [](file:///c%3A/vscode_src/zipgateway-main/src/ZIP_Router.c#900-900), lines 900 to 1060

Starting (3/6) *Map packet receive flow*

Completed (3/6) *Map packet receive flow*

Hoàn thành trace! Dưới đây là flow chi tiết từ boot đến xử lý packet, với code trích dẫn đầy đủ:

---

# **PHẦN 1: KHỞI ĐỘNG CONTIKI - Process Startup Sequence**

## **1.1. Boot Sequence trong main() (`contiki-main.c:103-143`)**

```c
int main(int argc, char** argv)
{
  signal(SIGINT, exit_handler);
  signal(SIGUSR1, sigusr1_handler);
  signal(SIGTERM, exit_handler);
  signal(SIGHUP, exit_handler);

  ConfigInit();   // Đọc zipgateway.cfg

  LOG_PRINTF("[HIEU] Starting Contiki main()\n");
  set_landev_outputfunc(tapdev_send);

  // ===== BƯỚC 1: Khởi tạo kernel =====
  process_init();     // Init process subsystem
  ctimer_init();      // Init callback timer

  // ===== BƯỚC 2: Start etimer_process =====
  procinit_init();    // Starts etimer_process với PROCESS_EVENT_INIT
  
  // ===== BƯỚC 3: Start tapdev_process =====
  serial_line_init();
  process_start(&tapdev_process, 0);  // Gọi đồng bộ PROCESS_EVENT_INIT
  LOG_PRINTF("[HIEU] contiki-main: process_start(&tapdev_process)\n");

  // ===== BƯỚC 4: Start zip_process =====
  autostart_start(autostart_processes);  // Chỉ có zip_process trong autostart

  setvbuf(stdout, (char *)NULL, _IONBF, 0);

  // ===== BƯỚC 5: Main event loop =====
  while(1) {
    fd_set fds;
    struct timeval tv;

    // Xử lý tất cả pending events và polls
    while (process_run()) {  // Loop cho đến khi hết event/poll
    }

    // Bail out nếu zip_process dừng
    if (!process_is_running(&zip_process)) {
      break;
    }

    // Tính etimer timeout
    clock_time_t delay = etimer_next_expiration_time();
    if (delay < clock_time()) delay = 100;
    else if (delay) delay -= clock_time();
    else delay = 200;

    // Setup file descriptors
    FD_ZERO(&fds);
    FD_SET(STDIN_FILENO, &fds);
    FD_SET(serial_fd, &fds);
    FD_SET(net_fd, &fds);  // TAP device file descriptor!
    
    n = serial_fd > net_fd ? serial_fd : net_fd;
    tv.tv_sec = delay / 1000;
    tv.tv_usec = (delay % 1000) * 1000;

    // ===== Chờ I/O hoặc timeout =====
    n = select(n + 1, &fds, NULL, NULL, &tv);
    
    if (FD_ISSET(net_fd, &fds)) {
      // TAP có data → trigger poll!
      process_poll(&tapdev_process);
    }
    if (FD_ISSET(serial_fd, &fds)) {
      serial_line_input_byte(...)  // Z-Wave serial data
    }
  }
}
```

### **Thứ tự khởi động process lần đầu**:

| Bước | Process | Event | Code location | Mục đích |
|------|---------|-------|---------------|----------|
| 1 | `etimer_process` | `PROCESS_EVENT_INIT` | `procinit_init()` → `contiki-main.c:123` | Timer subsystem |
| 2 | `tapdev_process` | `PROCESS_EVENT_INIT` | `process_start(&tapdev_process)` → `contiki-main.c:126` | TAP network interface |
| 3 | `zip_process` | `PROCESS_EVENT_INIT` | `autostart_start()` → `contiki-main.c:129` | ZIP Gateway orchestrator |

---

## **1.2. tapdev_process INIT Handler (`tapdev-drv.c:118-143`)**

```c
PROCESS_THREAD(tapdev_process, ev, data)
{
  PROCESS_POLLHANDLER(pollhandler());  // Chạy khi ev == PROCESS_EVENT_POLL

  PROCESS_BEGIN();

  // ===== Xử lý PROCESS_EVENT_INIT =====
  tapdev_init();  // Mở /dev/net/tun, init buffer

  tcpip_set_outputfunc(tapdev_send);  // Đăng ký output function
  LOG_PRINTF("[HIEU] tapdev-drv: tcpip_set_outputfunc(tapdev_send)\n");

  // Tự trigger poll ngay lập tức
  process_poll(&tapdev_process);

  // Setup ARP timer (10s)
  etimer_set(&arptimer, 10*1000);

  while (1) {
    PROCESS_YIELD();  // Chặn tại đây, chờ event tiếp theo
    
    if(ev == PROCESS_EVENT_TIMER) {
      uip_arp_timer();
      etimer_restart(&arptimer);
    }
  }

  tapdev_exit();
  PROCESS_END();
}
```

**Kết quả INIT**: TAP device sẵn sàng, poll flag được set.

---

## **1.3. zip_process INIT Handler (`ZIP_Router.c:1329-1545`)**

```c
PROCESS_THREAD(zip_process, ev, data)
{
  int i;
  PROCESS_BEGIN();

  LOG_PRINTF("Starting " PACKAGE_STRING " build " PACKAGE_SVN_REVISION "\n");

#ifdef NO_ZW_NVM
  zw_appl_nvm_init();
#endif

  while(1) {
    DBG_PRINTF("Event ***************** %x ********************\n", ev);
    
    // ===== Xử lý PROCESS_EVENT_INIT =====
    if (ev == PROCESS_EVENT_INIT) {
      if (!ZIP_Router_Reset()) {  // <--- Gọi init tổng!
        ERR_PRINTF("Fatal error\n");
        process_post(&zip_process, PROCESS_EVENT_EXIT, 0);
      }
    }
    // ... xử lý các event khác ...
    
    PROCESS_WAIT_EVENT();  // Chặn, chờ event tiếp theo
  }

  PROCESS_END();
}
```

---

## **1.4. ZIP_Router_Reset() - Khởi động tất cả server processes (`ZIP_Router.c:959-1060`)**

```c
static BOOL ZIP_Router_Reset()
{
  LOG_PRINTF("Resetting ZIP Gateway\n");

  // ===== 1. Serial API Process (Z-Wave module) =====
  process_exit(&serial_api_process);
  process_start(&serial_api_process, cfg.serial_port);  // → PROCESS_EVENT_INIT
  
  if (!serial_ok) return FALSE;

  MemoryGetID((BYTE*)&homeID, &MyNodeID);  // Lấy Home ID, Node ID từ Z-Wave
  
  // ===== 2. TCP/IP Stack Process =====
  process_exit(&tcpip_process);
  process_start(&tcpip_process, 0);  // → PROCESS_EVENT_INIT

  refresh_ipv6_addresses();
  
  // ===== 3. UDP Server Process (port 4123) =====
  process_exit(&udp_server_process);
  process_start(&udp_server_process, 0);  // → PROCESS_EVENT_INIT
  
  // ===== 4. DTLS Server Process (port 41230) =====
#ifndef DISABLE_DTLS
  process_exit(&dtls_server_process);
  process_start(&dtls_server_process, 0);  // → PROCESS_EVENT_INIT
#endif

  // ===== 5. mDNS Server Process (port 5353) =====
#ifdef SUPPORTS_MDNS
  process_start(&mDNS_server_process, 0);  // → PROCESS_EVENT_INIT
#endif

  // ===== 6. CoAP Server (optional) =====
#ifdef SUPPORTS_COAP
  process_exit(&coap_server_process);
  process_start(&coap_server_process, 0);
#endif

  // ===== 7. IPv4 Support (nếu enable) =====
#ifdef IP_4_6_NAT
  if(!cfg.ipv4disable && !ipv4_init_once) {
    ipv4_interface_init();
    
    // TCP client cho portal
    process_exit(&zip_tcp_client_process);
    process_start(&zip_tcp_client_process, cfg.portal_url);  // → INIT
    
    ipv4_init_once = 1;
    
    // DNS resolver
    process_exit(&resolv_process);
    process_start(&resolv_process, NULL);  // → INIT
    
    // NTP client
    process_exit(&ntpclient_process);
    process_start(&ntpclient_process, NULL);  // → INIT
  }
#endif

  // ===== 8. Data Store, Mailbox, Transport Service =====
  ZW_TransportService_Init(ApplicationCommandHandlerZIP);
  ZW_SendRequest_init();
  if (!data_store_init()) return FALSE;
  mb_init();

  // ===== 9. DHCP Client Process (port 68) =====
  process_exit(&dhcp_client_process);
  process_start(&dhcp_client_process, 0);  // → PROCESS_EVENT_INIT

  provisioning_list_init(...);

  // ===== 10. Network Management (async) =====
  network_management_init_done = 0;
  
  return TRUE;
}
```

### **Tổng kết process startup sau ZIP_Router_Reset()**:

| STT | Process | Event | Port | Chức năng |
|-----|---------|-------|------|-----------|
| 1 | `serial_api_process` | `INIT` | - | Giao tiếp với Z-Wave module qua serial |
| 2 | `tcpip_process` | `INIT` | - | TCP/IP stack core |
| 3 | `udp_server_process` | `INIT` | 4123 | Nhận Z/IP commands (không mã hóa) |
| 4 | `dtls_server_process` | `INIT` | 41230 | Nhận Z/IP commands (DTLS mã hóa) |
| 5 | `mDNS_server_process` | `INIT` | 5353 | Service discovery (multicast DNS) |
| 6 | `zip_tcp_client_process` | `INIT` | - | Kết nối portal server (optional) |
| 7 | `resolv_process` | `INIT` | - | DNS client |
| 8 | `ntpclient_process` | `INIT` | - | NTP time sync |
| 9 | `dhcp_client_process` | `INIT` | 68 | DHCP client (lấy IPv4) |

**Sau khi startup hoàn tất**: Tất cả processes đã YIELD, chờ events. Main loop đang chờ I/O trong `select()`.

---

# **PHẦN 2: XỬ LÝ GÓI TIN ĐẾN - Packet Processing Flow**

## **2.1. Packet Arrival → Poll Trigger (`contiki-main.c:178-180`)**

```c
// Main loop đã block tại select()
n = select(n + 1, &fds, NULL, NULL, &tv);

// Khi có frame mới trên TAP device:
if (FD_ISSET(net_fd, &fds)) {
  // Đánh dấu tapdev_process cần poll
  process_poll(&tapdev_process);
  // Đặt needspoll=1, poll_requested=1
}

// Quay lại đầu while(1) → process_run()
```

---

## **2.2. Scheduler Chạy Poll → POLLHANDLER (`process.c:317-327`)**

```c
int process_run(void)
{
  // ===== BƯỚC 1: Xử lý poll trước =====
  if(poll_requested) {
    do_poll();  // Gọi tất cả process có needspoll=1
  }

  // ===== BƯỚC 2: Xử lý 1 event từ queue =====
  do_event();

  return nevents + poll_requested;
}
```

### **do_poll() (`process.c:240-253`)**

```c
static void do_poll(void)
{
  struct process *p;
  
  poll_requested = 0;
  
  // Duyệt tất cả processes
  for(p = process_list; p != NULL; p = p->next) {
    if(p->needspoll) {
      p->state = PROCESS_STATE_RUNNING;
      p->needspoll = 0;
      
      // Gọi với PROCESS_EVENT_POLL
      call_process(p, PROCESS_EVENT_POLL, NULL);
    }
  }
}
```

---

## **2.3. tapdev_process Xử lý POLL → Đọc Frame (`tapdev-drv.c:73-121`)**

```c
static void pollhandler(void)
{
  // ===== Đọc frame từ TAP =====
  uip_len = tapdev_poll();  // Read from /dev/net/tun

  if(uip_len > 0) {
    LOG_PRINTF("[HIEU] tapdev-drv: got frame len=%d ethertype=0x%04x\n", 
               uip_len, UIP_HTONS(BUF->type));

    // ===== Phân loại theo EtherType =====
    
    if(BUF->type == uip_htons(UIP_ETHTYPE_IP)) {
      ipv46nat_interface_input();  // IPv4 NAT prep
    }

    if(BUF->type == uip_htons(UIP_ETHTYPE_IPV6)) {
      // Đánh dấu nguồn: LAN
      UIP_IP_BUF->flow = FLOW_FROM_LAN;
      
      LOG_PRINTF("[HIEU] tapdev-drv: dispatching IPv6 -> tcpip_input()\n");
      
      // ===== GỌI TCPIP_INPUT - ĐIỂM VÀO STACK! =====
      tcpip_input();
      
    } else if(BUF->type == uip_htons(UIP_ETHTYPE_IP)) {
      uip_len -= sizeof(struct uip_eth_hdr);
      LOG_PRINTF("[HIEU] tapdev-drv: dispatching IPv4 -> tcpip_ipv4_input()\n");
      tcpip_ipv4_input();
      
    } else if(BUF->type == uip_htons(UIP_ETHTYPE_ARP)) {
      uip_arp_arpin();
      if(uip_len > 0) {
        LOG_PRINTF("[HIEU] tapdev-drv: ARP reply prepared -> do_send()\n");
        do_send();
      }
    } else {
      uip_len = 0;  // Drop unknown frame
    }
  }
}
```

**Event flow tại đây**: 
- Event: `PROCESS_EVENT_POLL`
- Process: `tapdev_process`
- Handler: `PROCESS_POLLHANDLER(pollhandler())`

---

## **2.4. tcpip_input() → tcpip_process (`tcpip.c:584-602`)**

```c
void tcpip_input(void)
{
  // ===== Post ĐỒNG BỘ đến tcpip_process =====
  process_post_synch(&tcpip_process, PACKET_INPUT, NULL);
  
  uip_len = 0;
  uip_ext_len = 0;
}
```

**QUAN TRỌNG**: `process_post_synch` gọi TRỰC TIẾP thread của `tcpip_process`, KHÔNG qua queue!

---

## **2.5. tcpip_process Nhận PACKET_INPUT (`tcpip.c:577-579`)**

```c
PROCESS_THREAD(tcpip_process, ev, data)
{
  PROCESS_BEGIN();
  
  while(1) {
    PROCESS_YIELD();
    
    // ===== Xử lý PACKET_INPUT =====
    if(ev == PACKET_INPUT) {
      packet_input();  // <--- Gọi uIP stack!
    }
    
    // ... xử lý UDP_POLL, TCP_POLL, TIMER ...
  }
  
  PROCESS_END();
}
```

**Event flow**:
- Event: `PACKET_INPUT`
- Process: `tcpip_process`
- Handler: `packet_input()`

---

## **2.6. packet_input() → uip_input() (`tcpip.c:229-269`)**

```c
static void packet_input(void)
{
  if(uip_len > 0) {
    check_for_tcp_syn();
    
    // ===== GỌI UIP STACK =====
    uip_input();  // Macro: uip_process(UIP_DATA)
    
    // Nếu có output (reply packet)
    if(uip_len > 0) {
      tcpip_ipv6_output();  // Routing & neighbor discovery
    }
  }
}
```

---

## **2.7. uip_process(UIP_DATA) - Core Stack (`uip6.c:882-1500`)**

```c
void uip_process(u8_t flag)
{
  // ===== BƯỚC 1: Kiểm tra địa chỉ IP =====
  if(uip_is_addr_mcast(&UIP_IP_BUF->srcipaddr)) {
    goto drop;  // Source không được là multicast
  }

  // ===== BƯỚC 2: Routing decision =====
#if UIP_CONF_ROUTER
  if(!uip_ds6_is_my_addr(&UIP_IP_BUF->destipaddr) &&
     !uip_ds6_is_my_maddr(&UIP_IP_BUF->destipaddr)) {
    // Không phải địa chỉ của mình → forward
    if(!uip_is_addr_mcast(&UIP_IP_BUF->destipaddr) && ...) {
      // Kiểm tra TTL, MTU
      if(UIP_IP_BUF->ttl <= 1) {
        uip_icmp6_error_output(ICMP6_TIME_EXCEEDED, ...);
        goto send;
      }
      UIP_IP_BUF->ttl--;
      PRINTF("Forwarding packet...\n");
      goto send;  // Forward ra interface khác
    } else {
      goto drop;  // Link-local off-link
    }
  }
#endif

  // ===== BƯỚC 3: Extension headers =====
  uip_next_hdr = &UIP_IP_BUF->proto;
  uip_ext_len = 0;
  
  while(1) {
    switch(*uip_next_hdr) {
      case UIP_PROTO_TCP:
        goto tcp_input;
      
      case UIP_PROTO_UDP:
        goto udp_input;  // <--- Trường hợp của chúng ta!
      
      case UIP_PROTO_ICMP6:
        goto icmp6_input;
      
      case UIP_PROTO_HBHO:    // Hop-by-hop
      case UIP_PROTO_DESTO:   // Destination options
      case UIP_PROTO_ROUTING: // Routing header
      case UIP_PROTO_FRAG:    // Fragment
        // Parse extension header, update uip_next_hdr
        uip_next_hdr = &UIP_EXT_BUF->next;
        uip_ext_len += ...;
        break;
      
      case UIP_PROTO_NONE:
        goto drop;
    }
  }

// ===== BƯỚC 4: UDP Input =====
udp_input:
  PRINTF("Receiving UDP packet\n");
  
  // Checksum verification
  if(UIP_UDP_BUF->udpchksum != 0 && uip_udpchksum() != 0xffff) {
    goto drop;
  }
  
  // Port validation
  if(UIP_UDP_BUF->destport == 0) {
    goto drop;
  }

  // ===== DEMULTIPLEX: Tìm connection khớp =====
  for(uip_udp_conn = &uip_udp_conns[0];
      uip_udp_conn < &uip_udp_conns[UIP_UDP_CONNS];
      ++uip_udp_conn) {
    
    if(uip_udp_conn->lport != 0 &&                           // Port đã bind
       UIP_UDP_BUF->destport == uip_udp_conn->lport &&       // Dest port khớp
       (uip_udp_conn->rport == 0 ||                          // Remote port wildcard
        UIP_UDP_BUF->srcport == uip_udp_conn->rport) &&      // hoặc khớp src port
       (uip_is_addr_unspecified(&uip_udp_conn->ripaddr) ||   // Remote IP wildcard
        uip_ipaddr_cmp(&UIP_IP_BUF->srcipaddr, &uip_udp_conn->ripaddr))) {  // hoặc khớp src IP
      
      goto udp_found;  // FOUND!
    }
  }
  
  // Không tìm thấy connection
  PRINTF("udp: no matching connection found\n");
  uip_icmp6_error_output(ICMP6_DST_UNREACH, ICMP6_DST_UNREACH_NOPORT, 0);
  goto send;

udp_found:
  PRINTF("In udp_found\n");
  
  uip_conn = NULL;
  uip_flags = UIP_NEWDATA;
  uip_sappdata = uip_appdata = &uip_buf[UIP_LLH_LEN + UIP_IPUDPH_LEN];
  uip_slen = 0;
  
  // ===== GỌI APPLICATION CALLBACK =====
  UIP_UDP_APPCALL();  // Macro → tcpip_uipcall()

  // ... udp_send logic ...
}
```

**Ví dụ cụ thể**:
- Gói đến `[2001:db8::1]:4123` → Tìm thấy `udp_server_process` (port 4123)
- Gói đến `[2001:db8::1]:41230` → Tìm thấy `dtls_server_process` (port 41230)

---

## **2.8. tcpip_uipcall() → Deliver to Application (`tcpip.c:995-1040`)**

```c
void tcpip_uipcall(void)
{
  register uip_udp_appstate_t *ts;

  // ===== Lấy appstate từ connection =====
  if(uip_conn != NULL) {
    ts = &uip_conn->appstate;      // TCP
  } else {
    ts = &uip_udp_conn->appstate;  // UDP (trường hợp này!)
  }

#if UIP_TCP
  // Xử lý TCP listen port (nếu là connection mới)
  if(uip_connected()) {
    for(i = 0; i < UIP_LISTENPORTS; ++i) {
      if(s.listenports[i].port == uip_conn->lport &&
         s.listenports[i].p != PROCESS_NONE) {
        ts->p = s.listenports[i].p;  // Gán process!
        ts->state = NULL;
        break;
      }
    }
    start_periodic_tcp_timer();
  }
#endif

  // ===== POST EVENT ĐẾN APPLICATION PROCESS =====
  if(ts->p != NULL) {
    process_post_synch(ts->p, tcpip_event, ts->state);
    // ts->p = &udp_server_process hoặc &dtls_server_process!
  }
}
```

**Event flow**:
- Event: `tcpip_event`
- Process: `udp_server_process` (nếu port 4123) hoặc `dtls_server_process` (nếu 41230)
- Data: `ts->state` (user appstate, thường là `&server_conn`)

---

## **2.9. udp_server_process Xử lý Packet (`ZW_udp_server.c:1214-1223`)**

```c
PROCESS_THREAD(udp_server_process, ev, data)
{
  PROCESS_BEGIN();
  
  // ... INIT: udp_bind(server_conn, ZWAVE_PORT) ...
  
  while (1) {
    PROCESS_YIELD();
    
    // ===== Xử lý tcpip_event =====
    if (ev == tcpip_event) {
      if (data == &server_conn) {  // Kiểm tra connection pointer
        
        // ===== XỬ LÝ Z/IP COMMAND =====
        tcpip_handler();
        
        // tcpip_handler() sẽ:
        // 1. Parse Z/IP header (command, seq, flags, options)
        // 2. Kiểm tra encryption bit → forward to DTLS nếu cần
        // 3. Kiểm tra session (source IP+port)
        // 4. Gọi ApplicationCommandHandlerZIP()
        // 5. Route command đến Z-Wave module hoặc local node
        // 6. Xử lý response, gửi reply packet
      }
    }
  }
  
  PROCESS_END();
}
```

---

## **2.10. dtls_server_process Xử lý DTLS Packet (`DTLS_server.c:547-620`)**

```c
PROCESS_THREAD(dtls_server_process, ev, data)
{
  PROCESS_BEGIN();
  
  // ... INIT: SSL_library_init(), udp_bind(server_conn, DTLS_PORT) ...
  
  while (1) {
    PROCESS_WAIT_EVENT();
    
    // ===== Xử lý tcpip_event =====
    if ((ev == tcpip_event && uip_newdata()) || (ev == DTLS_SERVER_INPUT_EVENT)) {
      struct uip_udp_conn *c = get_udp_conn();
      
      // Kiểm tra content type: 0x16 = Handshake
      if(((u8_t*)uip_appdata)[0] == 0x16) {
        DBG_PRINTF("DTLS input is handshake:\n");
        // Parse handshake type: ClientHello, ServerHello, etc.
      }
      
      // ===== Tìm hoặc tạo DTLS session =====
      struct dtls_session *s = dtls_find_session(c);
      if(!s) {
        // Tạo session mới cho client này
        s = dtls_new_session(c);
        if(!s) goto drop;
        
        // Setup SSL object
        s->ssl = SSL_new(ctx);
        SSL_set_bio(s->ssl, bio_read, bio_write);
      }
      
      // ===== DTLS Decryption =====
      len = SSL_read(s->ssl, read_buf, sizeof(read_buf));
      
      if(len > 0) {
        // Decrypt thành công → forward to Z/IP handler
        memcpy(uip_appdata, read_buf, len);
        uip_len = len;
        
        // Post event để xử lý decrypted data
        process_post(&udp_server_process, DTLS_SERVER_INPUT_EVENT, s);
      } else {
        // Handshake in progress hoặc error
        int err = SSL_get_error(s->ssl, len);
        if(err == SSL_ERROR_WANT_READ) {
          // Cần thêm data, chờ packet tiếp theo
        }
      }
    }
  }
  
  PROCESS_END();
}
```

---

# **PHẦN 3: FLOW DIAGRAM TỔNG HỢP**

## **3.1. Startup Flow (Boot to Ready)**

```
main()
  │
  ├─> process_init()                    // Init kernel
  ├─> ctimer_init()
  ├─> procinit_init()                   
  │    └─> process_start(&etimer_process)
  │         └─> [PROCESS_EVENT_INIT] → etimer_process
  │
  ├─> process_start(&tapdev_process)
  │    └─> [PROCESS_EVENT_INIT] → tapdev_process
  │         ├─> tapdev_init()           // Open TAP
  │         ├─> tcpip_set_outputfunc()  // Register output
  │         ├─> process_poll(&tapdev_process)  // Self-poll
  │         ├─> etimer_set(&arptimer)
  │         └─> PROCESS_YIELD()         // Wait for POLL
  │
  ├─> autostart_start(autostart_processes)
  │    └─> process_start(&zip_process)
  │         └─> [PROCESS_EVENT_INIT] → zip_process
  │              └─> ZIP_Router_Reset()
  │                   ├─> process_start(&serial_api_process)      → [INIT]
  │                   ├─> process_start(&tcpip_process)           → [INIT]
  │                   ├─> process_start(&udp_server_process)      → [INIT]
  │                   │    └─> udp_bind(server_conn, 4123)
  │                   ├─> process_start(&dtls_server_process)     → [INIT]
  │                   │    └─> udp_bind(server_conn, 41230)
  │                   ├─> process_start(&mDNS_server_process)     → [INIT]
  │                   │    └─> udp_bind(server_conn, 5353)
  │                   ├─> process_start(&zip_tcp_client_process)  → [INIT]
  │                   ├─> process_start(&resolv_process)          → [INIT]
  │                   ├─> process_start(&ntpclient_process)       → [INIT]
  │                   ├─> process_start(&dhcp_client_process)     → [INIT]
  │                   │    └─> udp_bind(conn, 68)
  │                   └─> data_store_init(), mb_init(), ...
  │
  └─> while(1) {
       while(process_run()) {}  // Xử lý tất cả pending events
       select(fds)               // Chờ I/O hoặc timer
      }

===== TẤT CẢ PROCESSES ĐÃ YIELD, SYSTEM READY =====
```

---

## **3.2. Packet Processing Flow (Frame to Application)**

```
┌─────────────────────────────────────────────────────────────────┐
│ ETHERNET FRAME ĐẾN TAP DEVICE                                   │
└─────────────────────────────────────────────────────────────────┘
  │
  ├─> select() phát hiện net_fd readable
  │    └─> process_poll(&tapdev_process)
  │         └─> needspoll=1, poll_requested=1
  │
  ├─> process_run()
  │    └─> do_poll()
  │         └─> [PROCESS_EVENT_POLL] → tapdev_process
  │              └─> PROCESS_POLLHANDLER(pollhandler())
  │                   ├─> uip_len = tapdev_poll()  // Đọc frame
  │                   ├─> Kiểm tra EtherType:
  │                   │    ├─> IPv6 → tcpip_input()
  │                   │    ├─> IPv4 → tcpip_ipv4_input()
  │                   │    └─> ARP  → uip_arp_arpin()
  │                   │
  │                   └─> tcpip_input()
  │                        └─> process_post_synch(&tcpip_process, PACKET_INPUT, NULL)
  │
  ├─> [PACKET_INPUT] → tcpip_process (SYNCHRONOUS CALL!)
  │    └─> packet_input()
  │         └─> uip_input()  // uip_process(UIP_DATA)
  │              ├─> Kiểm tra địa chỉ IP (src/dst)
  │              ├─> Routing decision (local/forward)
  │              ├─> Parse extension headers (HBHO, Routing, Fragment...)
  │              └─> switch(proto):
  │                   ├─> UIP_PROTO_TCP    → tcp_input
  │                   ├─> UIP_PROTO_UDP    → udp_input
  │                   └─> UIP_PROTO_ICMP6  → icmp6_input
  │
  ├─> udp_input:
  │    ├─> Verify checksum
  │    ├─> Validate port != 0
  │    ├─> for(uip_udp_conn in uip_udp_conns[]) {
  │    │     if(destport == lport &&
  │    │        (rport == 0 || srcport == rport) &&
  │    │        (ripaddr unspecified || srcip == ripaddr)) {
  │    │       goto udp_found;  // MATCH!
  │    │     }
  │    │   }
  │    │
  │    ├─> Không tìm thấy:
  │    │    └─> uip_icmp6_error_output(ICMP6_DST_UNREACH, NOPORT)
  │    │
  │    └─> udp_found:
  │         ├─> uip_flags = UIP_NEWDATA
  │         ├─> uip_appdata = &uip_buf[IPUDPH_LEN]
  │         └─> UIP_UDP_APPCALL()  // tcpip_uipcall()
  │
  ├─> tcpip_uipcall()
  │    ├─> ts = &uip_udp_conn->appstate
  │    │     └─> ts->p = &udp_server_process (nếu port 4123)
  │    │         hoặc &dtls_server_process (nếu port 41230)
  │    │         hoặc &mDNS_server_process (nếu port 5353)
  │    │
  │    └─> process_post_synch(ts->p, tcpip_event, ts->state)
  │
  └─> [tcpip_event] → udp_server_process (nếu port 4123)
       │
       └─> if (ev == tcpip_event && data == &server_conn) {
            tcpip_handler()
            ├─> Parse Z/IP header
            ├─> Kiểm tra encryption bit
            ├─> Session management
            ├─> ApplicationCommandHandlerZIP()
            │    ├─> Route to Z-Wave module (serial)
            │    ├─> Hoặc xử lý local (gateway node)
            │    └─> Command class handlers
            └─> Send reply packet
           }

HOẶC

  └─> [tcpip_event] → dtls_server_process (nếu port 41230)
       │
       └─> if ((ev == tcpip_event && uip_newdata()) || ...) {
            ├─> dtls_find_session(c)
            ├─> Nếu handshake (0x16):
            │    ├─> dtls_new_session()
            │    ├─> SSL_new(), SSL_set_bio()
            │    └─> SSL_accept() / SSL_read()
            │
            ├─> SSL_read(s->ssl, read_buf, size)
            │    └─> Decrypt DTLS packet
            │
            └─> Forward decrypted data:
                 process_post(&udp_server_process, DTLS_SERVER_INPUT_EVENT, s)
           }
```

---

## **3.3. Event Summary Table**

| Event | Source | Target Process | Handler | Timing |
|-------|--------|----------------|---------|--------|
| `PROCESS_EVENT_INIT` | `process_start()` | Any new process | `if(ev == INIT)` hoặc code trước `while(1)` | Synchronous |
| `PROCESS_EVENT_POLL` | `process_poll()` → `do_poll()` | `tapdev_process` | `PROCESS_POLLHANDLER(pollhandler())` | Priority (before events) |
| `PACKET_INPUT` | `tcpip_input()` | `tcpip_process` | `packet_input()` | Synchronous |
| `tcpip_event` | `tcpip_uipcall()` | `udp_server_process`, `dtls_server_process`, etc. | `if(ev == tcpip_event)` | Synchronous |
| `DTLS_SERVER_INPUT_EVENT` | `dtls_server_process` | `udp_server_process` | `tcpip_handler()` | Async (queued) |
| `PROCESS_EVENT_TIMER` | `etimer` expiry | Process với etimer đang chạy | `if(ev == TIMER)` | Async (queued) |
| `PROCESS_EVENT_EXIT` | `process_exit()` | Target process | `PROCESS_EXITHANDLER()` | Synchronous |
| `PROCESS_EVENT_EXITED` | `exit_process()` | All other processes (broadcast) | `if(ev == EXITED)` | Synchronous (each) |

---

## **3.4. Key Takeaways**

1. **Khởi động tuần tự**: `etimer` → `tapdev` → `zip_process` → tất cả server processes.
2. **Mỗi process nhận INIT đồng bộ** khi được `process_start()`.
3. **Poll có ưu tiên cao nhất**, chạy trước event queue.
4. **Packet flow đồng bộ hoàn toàn** từ TAP → pollhandler → `tcpip_input()` → `uip_process()` → `tcpip_uipcall()` → application.
5. **Demux theo port**: Linear search trong `uip_udp_conns[]`, so khớp `lport`, `rport`, `ripaddr`.
6. **Mỗi connection lưu con trỏ process** trong `appstate.p`, được set khi gọi `udp_new()`.
7. **DTLS decrypt rồi forward** sang `udp_server_process` để xử lý Z/IP command.

Bạn muốn trace sâu hơn vào phần nào? Ví dụ:
- Chi tiết `tcpip_handler()` xử lý Z/IP command?
- Flow output (reply packet) từ application → TAP?
- DTLS handshake sequence chi tiết?
- Network Management state machine?
