Tôi sẽ trace toàn bộ flow xử lý gói tin từ `tcpip_input()` cho bạn, bắt đầu từ file định nghĩa `tcpip_process`. 

Starting (1/5) *Trace tcpip_input entry point*

Searched text for `^tcpip_input\(|PROCESS_THREAD\(tcpip_process|PACKET_INPUT|packet_input\(` (`**/contiki/core/net/tcpip.c`), 18 results

Read [](file:///c%3A/vscode_src/zipgateway-main/contiki/core/net/tcpip.c#584-584), lines 584 to 610

Read [](file:///c%3A/vscode_src/zipgateway-main/contiki/core/net/tcpip.c#229-229), lines 229 to 280

Read [](file:///c%3A/vscode_src/zipgateway-main/contiki/core/net/tcpip.c#570-570), lines 570 to 590

Searched text for `^uip_input\(|uip_process\(|UIP_PROTO_` (`**/contiki/core/net/uip6.c`), 20 results

Searched text for `^uip_input\(\)|#define uip_input` (`**/contiki/core/net/uip6.c`), no results

Searched text for `define uip_input|uip_input\(\)` (`**/contiki/core/net/*.h`), 9 results

Read [](file:///c%3A/vscode_src/zipgateway-main/contiki/core/net/uip6.c#880-880), lines 880 to 940

Read [](file:///c%3A/vscode_src/zipgateway-main/contiki/core/net/uip6.c#1100-1100), lines 1100 to 1200

Starting (3/5) *Map uIP stack flow*

Searched text for `udp_input:|tcp_input:|icmp6_input:|UIP_APPCALL|tcpip_uipcall` (`**/contiki/core/net/uip6.c`), 20 results

Read [](file:///c%3A/vscode_src/zipgateway-main/contiki/core/net/uip6.c#1442-1442), lines 1442 to 1520

Searched text for `#define UIP_APPCALL|#define UIP_UDP_APPCALL|tcpip_uipcall` (`**/contiki/core/net/tcpip.c`), 2 results

Read [](file:///c%3A/vscode_src/zipgateway-main/contiki/core/net/tcpip.c#995-995), lines 995 to 1040

Searched text for `UIP_APPCALL|UIP_UDP_APPCALL` (`**/contiki/core/net/tcpip.h`), 2 results

Starting (5/5) *Map output/forwarding path*

Searched text for `tcpip_ipv6_output|tcpip_output\(|uip_nd6_ns_output|uip_nd6_na_output` (`**/contiki/core/net/tcpip.c`), 20 results

Read [](file:///c%3A/vscode_src/zipgateway-main/contiki/core/net/tcpip.c#607-607), lines 607 to 700

Read [](file:///c%3A/vscode_src/zipgateway-main/contiki/core/net/tcpip.c#700-700), lines 700 to 780

Read [](file:///c%3A/vscode_src/zipgateway-main/contiki/core/net/tcpip.c#780-780), lines 780 to 830

Completed (5/5) *Map output/forwarding path*

Tôi đã trace xong toàn bộ flow xử lý gói tin từ `tcpip_input()`. Dưới đây là bản đồ chi tiết với code trích dẫn:

## **Flow tổng quát từ pollhandler → ứng dụng**

```
pollhandler()
  ↓ tapdev_poll() → đọc frame từ TAP
  ↓ phân loại EtherType (IPv6/IPv4/ARP)
  ↓
tcpip_input()                         [tcpip.c:584]
  ↓ process_post_synch(&tcpip_process, PACKET_INPUT, NULL)  [đồng bộ!]
  ↓
tcpip_process nhận PACKET_INPUT       [tcpip.c:577]
  ↓ case PACKET_INPUT: packet_input()
  ↓
packet_input()                        [tcpip.c:229]
  ↓ uip_input()  → macro: uip_process(UIP_DATA)  [uip.h:323]
  ↓
uip_process(UIP_DATA)                 [uip6.c:882]
  ↓ kiểm tra routing / forwarding
  ↓ xử lý extension headers (HBHO, Routing, Fragment...)
  ↓ switch(proto):
     ├─ UIP_PROTO_TCP → tcp_input
     ├─ UIP_PROTO_UDP → udp_input
     └─ UIP_PROTO_ICMP6 → icmp6_input
  ↓
udp_input / tcp_input                 [uip6.c:1442 / 1562]
  ↓ tìm connection (uip_udp_conn / uip_conn)
  ↓ UIP_UDP_APPCALL() / UIP_APPCALL()
  ↓   → macro: tcpip_uipcall()       [tcpip.h:79-80]
  ↓
tcpip_uipcall()                       [tcpip.c:995]
  ↓ lấy appstate (ts = conn->appstate)
  ↓ process_post_synch(ts->p, tcpip_event, ts->state)
  ↓
[Ứng dụng (UDP/TCP server)] nhận tcpip_event  [ví dụ: udp_server_process, dtls_server_process...]
  ↓ xử lý data trong uip_appdata
  ↓ gọi uip_send() nếu cần trả lời
  ↓
packet_input() tiếp tục               [tcpip.c:258]
  ↓ if(uip_len > 0) → có reply
  ↓   tcpip_ipv6_output()             [tcpip.c:607 hoặc 815]
  ↓
tcpip_ipv6_output()                   
  ↓ kiểm tra MTU, địa chỉ unspecified
  ↓ routing decision:
     ├─ multicast → tcpip_output(0)
     ├─ onlink → nexthop = destaddr
     ├─ source routing (RPL)
     ├─ route table lookup → nexthop
     └─ default route
  ↓ neighbor discovery:
     ├─ nbr not in cache → uip_ds6_nbr_add() → uip_nd6_ns_output() (gửi NS)
     ├─ nbr incomplete → queue packet
     ├─ nbr stale → move to DELAY
     └─ nbr reachable → tcpip_output(&nbr->lladdr)
  ↓
tcpip_output(&lladdr)                 [tcpip.c:159 hoặc 182]
  ↓ gọi tcpip_output_func (đã set bởi tapdev/zwave)
  ↓   → tapdev_send() hoặc ZIP_output()
  ↓
Gói tin ra ngoài (TAP hoặc Z-Wave)
```

---

## **1. Điểm vào: tcpip_input() (`tcpip.c:584`)**

```c
void tcpip_input(void)
{
  process_post_synch(&tcpip_process, PACKET_INPUT, NULL);  // Gọi đồng bộ!
  uip_len = 0;
  uip_ext_len = 0;
}
```

**Quan trọng**: Đây là **synchronous call**, không qua hàng đợi! Nghĩa là `tcpip_process` chạy ngay trong ngữ cảnh `pollhandler()`.

---

## **2. tcpip_process xử lý PACKET_INPUT (`tcpip.c:577`)**

```c
PROCESS_THREAD(tcpip_process, ev, data)
{
  PROCESS_BEGIN();
  ...
  while(1) {
    PROCESS_YIELD();
    
    if(ev == PACKET_INPUT) {
      packet_input();
    }
    ...
  }
  PROCESS_END();
}
```

---

## **3. packet_input() → uip_input() (`tcpip.c:229`)**

```c
static void packet_input(void)
{
  if(uip_len > 0) {
    check_for_tcp_syn();
    uip_input();              // Macro: uip_process(UIP_DATA)
    if(uip_len > 0) {         // Nếu có output (reply)
      tcpip_ipv6_output();
    }
  }
}
```

---

## **4. uip_process(UIP_DATA) - Core của uIP stack (`uip6.c:882`)**

### 4.1. Kiểm tra địa chỉ và routing (`uip6.c:1100–1170`)

```c
void uip_process(u8_t flag)
{
  ...
  // Kiểm tra src không phải multicast
  if(uip_is_addr_mcast(&UIP_IP_BUF->srcipaddr)) {
    goto drop;
  }

#if UIP_CONF_ROUTER
  // Nếu dest không phải địa chỉ của mình → forward
  if(!uip_ds6_is_my_addr(&UIP_IP_BUF->destipaddr) &&
     !uip_ds6_is_my_maddr(&UIP_IP_BUF->destipaddr)) {
    if(!uip_is_addr_mcast(&UIP_IP_BUF->destipaddr) && ...) {
      // Kiểm tra TTL, MTU
      UIP_IP_BUF->ttl = UIP_IP_BUF->ttl - 1;
      goto send;  // Forward packet
    } else {
      goto drop;  // Link-local off-link → drop
    }
  }
#endif
  ...
```

### 4.2. Extension header processing (`uip6.c:1177–1340`)

```c
  uip_next_hdr = &UIP_IP_BUF->proto;
  uip_ext_len = 0;
  
  while(1) {
    switch(*uip_next_hdr) {
      case UIP_PROTO_TCP:
        goto tcp_input;
      case UIP_PROTO_UDP:
        goto udp_input;
      case UIP_PROTO_ICMP6:
        goto icmp6_input;
      case UIP_PROTO_HBHO:    // Hop-by-hop
      case UIP_PROTO_DESTO:   // Destination options
      case UIP_PROTO_ROUTING: // Routing header
      case UIP_PROTO_FRAG:    // Fragment
        // Xử lý từng loại extension header...
        uip_next_hdr = &UIP_EXT_BUF->next;
        uip_ext_len += ...;
        break;
      case UIP_PROTO_NONE:
        goto drop;
    }
  }
```

---

## **5. UDP Input → Delivery to App (`uip6.c:1442`)**

```c
udp_input:
  UIP_STAT(++uip_stat.udp.recv);
  
  // Kiểm tra checksum
  if(UIP_UDP_BUF->udpchksum != 0 && uip_udpchksum() != 0xffff) {
    goto drop;
  }
  
  // Tìm connection khớp (port + địa chỉ)
  for(uip_udp_conn = &uip_udp_conns[0];
      uip_udp_conn < &uip_udp_conns[UIP_UDP_CONNS];
      ++uip_udp_conn) {
    if(uip_udp_conn->lport != 0 &&
       UIP_UDP_BUF->destport == uip_udp_conn->lport &&
       (uip_udp_conn->rport == 0 || UIP_UDP_BUF->srcport == uip_udp_conn->rport) &&
       (uip_is_addr_unspecified(&uip_udp_conn->ripaddr) ||
        uip_ipaddr_cmp(&UIP_IP_BUF->srcipaddr, &uip_udp_conn->ripaddr))) {
      goto udp_found;
    }
  }
  
  // Không tìm thấy → ICMP port unreachable
  uip_icmp6_error_output(ICMP6_DST_UNREACH, ICMP6_DST_UNREACH_NOPORT, 0);
  goto send;

udp_found:
  uip_conn = NULL;
  uip_flags = UIP_NEWDATA;
  uip_sappdata = uip_appdata = &uip_buf[UIP_LLH_LEN + UIP_IPUDPH_LEN];
  uip_slen = 0;
  UIP_UDP_APPCALL();   // Macro → tcpip_uipcall()
  ...
```

---

## **6. tcpip_uipcall() - Giao packet cho process app (`tcpip.c:995`)**

```c
void tcpip_uipcall(void)
{
  register uip_udp_appstate_t *ts;
  
  if(uip_conn != NULL) {
    ts = &uip_conn->appstate;      // TCP
  } else {
    ts = &uip_udp_conn->appstate;  // UDP
  }
  
#if UIP_TCP
  // Xử lý TCP listen port (gán process nếu là connection mới)
  if(uip_connected()) {
    for(i = 0; i < UIP_LISTENPORTS; ++i) {
      if(s.listenports[i].port == uip_conn->lport && ...) {
        ts->p = s.listenports[i].p;
        ts->state = NULL;
        break;
      }
    }
  }
#endif
  
  if(ts->p != NULL) {
    process_post_synch(ts->p, tcpip_event, ts->state);  // Giao cho app!
  }
}
```

**Cơ chế**: Mỗi UDP/TCP connection có `appstate` chứa con trỏ process (`ts->p`). Khi có data mới, kernel post `tcpip_event` đến process đó (ví dụ `udp_server_process`, `dtls_server_process`, `coap_server_process`).

---

## **7. Output Path: tcpip_ipv6_output() (`tcpip.c:607 hoặc 815`)**

### 7.1. Routing decision

```c
void tcpip_ipv6_output(void)
{
  uip_ds6_nbr_t *nbr = NULL;
  uip_ipaddr_t* nexthop;
  
  // Kiểm tra MTU, địa chỉ unspecified
  if(uip_len > UIP_LINK_MTU) { ... return; }
  if(uip_is_addr_unspecified(&UIP_IP_BUF->destipaddr)) { ... return; }
  
  // IPv4-mapped → route đến interface đặc biệt
  if(uip_is_4to6_addr(&UIP_IP_BUF->destipaddr)) {
    tcpip_output(NULL);
    return;
  }
  
  // Multicast → gửi thẳng
  if(uip_is_addr_mcast(&UIP_IP_BUF->destipaddr)) {
    tcpip_output(0);
    return;
  }
  
  // On-link → nexthop = destination
  if(uip_ds6_is_addr_onlink(&UIP_IP_BUF->destipaddr)) {
    nexthop = &UIP_IP_BUF->destipaddr;
    goto nexthop_done;
  }
  
  // Source routing (RPL)
  sr = rpl_sr_get_route(&src, &dest);
  if(sr) {
    rpl_sr_add_hdr(sr);
    nexthop = &UIP_IP_BUF->destipaddr;
    goto nexthop_done;
  }
  
  // Route table lookup
  locrt = uip_ds6_route_lookup(&UIP_IP_BUF->destipaddr);
  if(locrt) {
    nexthop = &locrt->nexthop;
    goto nexthop_done;
  }
  
  // Default route
  nexthop = uip_ds6_defrt_choose();
  if(!nexthop) {
    PRINTF("Destination off-link but no route\n");
    uip_len = 0;
    return;
  }

nexthop_done:
  ...
```

### 7.2. Neighbor Discovery & Link-layer resolution

```c
nexthop_done:
  // Tìm neighbor trong cache
  if((nbr = uip_ds6_nbr_lookup(nexthop)) == NULL) {
    PRINTF("Neighbor not in cache.\n");
    
    // Thêm neighbor mới với state INCOMPLETE
    if((nbr = uip_ds6_nbr_add(nexthop, NULL, 0, NBR_INCOMPLETE)) == NULL) {
      uip_len = 0;
      return;
    }
    
    // Queue packet
#if UIP_CONF_IPV6_QUEUE_PKT
    uip_packetqueue_alloc(&nbr->packethandle, ...);
    memcpy(uip_packetqueue_buf(&nbr->packethandle), UIP_IP_BUF, uip_len);
#endif
    
    // Gửi Neighbor Solicitation
    if(uip_ds6_is_my_addr(&UIP_IP_BUF->srcipaddr)) {
      uip_nd6_ns_output(&UIP_IP_BUF->srcipaddr, NULL, &nbr->ipaddr);
    } else {
      uip_nd6_ns_output(NULL, NULL, &nbr->ipaddr);
    }
    
    stimer_set(&nbr->sendns, uip_ds6_if.retrans_timer / 1000);
    nbr->nscount = 1;
    
  } else {
    // Neighbor đã có trong cache
    if(nbr->state == NBR_PERMANTLY_UNREACHABLE) {
      uip_icmp6_error_output(ICMP6_DST_UNREACH, ICMP6_DST_UNREACH_ADDR, 0);
      return;
    }
    
    if(nbr->state == NBR_INCOMPLETE) {
      // Vẫn chờ NS reply → queue packet
#if UIP_CONF_IPV6_QUEUE_PKT
      uip_packetqueue_alloc(&nbr->packethandle, ...);
      memcpy(...);
#endif
      uip_len = 0;
      return;
    }
    
    if(nbr->state == NBR_STALE) {
      nbr->state = NBR_DELAY;
      stimer_set(&nbr->reachable, UIP_ND6_DELAY_FIRST_PROBE_TIME);
    }
    
    // Gửi packet với L2 address đã biết
    tcpip_output(&nbr->lladdr);
    
    // Gửi các packet đã queue (nếu có)
#if UIP_CONF_IPV6_QUEUE_PKT
    if(uip_packetqueue_buflen(&nbr->packethandle) != 0) {
      uip_len = uip_packetqueue_buflen(&nbr->packethandle);
      memcpy(UIP_IP_BUF, uip_packetqueue_buf(&nbr->packethandle), uip_len);
      uip_packetqueue_free(&nbr->packethandle);
      tcpip_output(&nbr->lladdr);
    }
#endif
    uip_len = 0;
    return;
  }
}
```

---

## **8. tcpip_output() - Gọi driver output (`tcpip.c:159`)**

```c
static u8_t tcpip_output(uip_lladdr_t *a)
{
  int ret;
  if(outputfunc != NULL) {
    ret = outputfunc(a);      // tapdev_send() hoặc ZIP_output()
    return ret;
  }
  UIP_LOG("tcpip_output: Use tcpip_set_outputfunc() to set an output function");
  return 0;
}
```

`outputfunc` được set bởi:
- `tapdev_process` INIT: `tcpip_set_outputfunc(tapdev_send);`
- Hoặc Z-Wave side: `tcpip_set_outputfunc(ZIP_output);`

---

## **Tóm tắt các bước then chốt**

| Bước | Hàm/File | Nhiệm vụ |
|------|----------|----------|
| **1. Entry** | `tcpip_input()` (tcpip.c:584) | Post đồng bộ PACKET_INPUT → tcpip_process |
| **2. Dispatch** | `packet_input()` (tcpip.c:229) | Gọi uip_input() (uip_process) |
| **3. Routing** | `uip_process()` (uip6.c:882) | Kiểm tra địa chỉ, forward/local, xử lý extension headers |
| **4. Demux** | `udp_input/tcp_input` (uip6.c:1442/1562) | Tìm connection, gọi UIP_APPCALL |
| **5. App delivery** | `tcpip_uipcall()` (tcpip.c:995) | Post tcpip_event đến process app (udp_server, dtls...) |
| **6. Output** | `tcpip_ipv6_output()` (tcpip.c:607) | Routing decision, neighbor discovery, queue NS |
| **7. TX** | `tcpip_output()` (tcpip.c:159) | Gọi driver (tapdev_send/ZIP_output) |

---

Bạn muốn mình trace sâu hơn vào phần nào? Ví dụ:
- ICMP6 handling (Router Solicitation/Advertisement, Neighbor Discovery)?
- TCP state machine chi tiết?
- Cách UDP server (ví dụ DTLS/CoAP) xử lý tcpip_event?
- Flow từ Z-Wave side ngược lại (ZIP_output → RF)?
