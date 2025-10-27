Tôi sẽ trace chi tiết hành trình của gói PING từ **192.168.0.100 → 192.168.0.104** qua tất cả các interface và các hàm xử lý:

---

# **HÀNH TRÌNH GÓI PING: 192.168.0.100 → 192.168.0.104 (Z-Wave Node)**

---

## **PHASE 1: PHYSICAL NETWORK - ETH0**

### **[ETH0] Packet arrives on Physical Ethernet**

```
┌─────────────────────────────────────────────────────────┐
│ PHYSICAL LAYER (eth0)                                   │
│ Gói tin ICMP Echo Request từ PC                         │
└─────────────────────────────────────────────────────────┘

ETHERNET FRAME:
┌──────────────────────────────────────────────────────────┐
│ Dest MAC:   AA:BB:CC:DD:EE:FF  (Gateway MAC)            │
│ Src MAC:    11:22:33:44:55:66  (PC MAC)                 │
│ EtherType:  0x0800 (IPv4)                                │
├──────────────────────────────────────────────────────────┤
│ IPv4 Header:                                             │
│   Version:        4                                      │
│   Protocol:       1 (ICMP)                               │
│   Src IP:         192.168.0.100                          │
│   Dst IP:         192.168.0.104  ← Node IPv4             │
│   TTL:            64                                     │
├──────────────────────────────────────────────────────────┤
│ ICMP Header:                                             │
│   Type:           8 (Echo Request)                       │
│   Code:           0                                      │
│   ID:             0x1234                                 │
│   Sequence:       1                                      │
│   Data:           "abcdefgh..."                          │
└──────────────────────────────────────────────────────────┘
```

**Interface:** `eth0` (Physical NIC)
**Action:** Nhận frame từ wire, đưa lên kernel network stack

---

## **PHASE 2: LINUX BRIDGE - BR-LAN**

### **[BR-LAN] Linux Bridge Processing**

**Interface:** `br-lan` (Linux Bridge)
**Code:** Linux kernel bridge code (`net/bridge/br_input.c`)

```c
// Kernel function: br_handle_frame()
// File: net/bridge/br_input.c

int br_handle_frame_finish(struct net_bridge_port *p, struct sk_buff *skb)
{
    // 1. Learn source MAC
    br_fdb_update(br, p, eth_hdr(skb)->h_source);
    
    // 2. Lookup destination MAC in FDB
    dst = br_fdb_find(br, eth_hdr(skb)->h_dest);
    
    // 3. Forward to destination port
    if (dst && dst->dst->dev) {
        // Unicast to specific port
        br_forward(dst->dst, skb);  // → Forward to tap0
    }
}
```

**Bridge FDB (Forwarding Database):**
```
MAC Address          Port      Age
AA:BB:CC:DD:EE:FF → tap0      0s   (Gateway MAC learned on tap0)
11:22:33:44:55:66 → eth0      0s   (PC MAC learned on eth0)
```

**Action:** 
- Learn PC MAC on eth0
- Lookup Gateway MAC → found on tap0
- Forward frame to tap0

**FRAME KHÔNG ĐỔI - chỉ chuyển port:**
```
eth0 → br-lan → tap0
```

---

## **PHASE 3: TAP DEVICE - TAP0**

### **[TAP0] Virtual TAP Interface**

**Interface:** `tap0` (Virtual network device)
**File descriptor:** `net_fd` (opened in tapdev_init)

```
┌─────────────────────────────────────────────────────────┐
│ TAP0 INTERFACE                                          │
│ Frame available in kernel buffer                        │
└─────────────────────────────────────────────────────────┘

FRAME TRÊN TAP0 (GIỐNG FRAME TỪ ETH0):
┌──────────────────────────────────────────────────────────┐
│ Dest MAC:   AA:BB:CC:DD:EE:FF                            │
│ Src MAC:    11:22:33:44:55:66                            │
│ EtherType:  0x0800 (IPv4)                                │
├──────────────────────────────────────────────────────────┤
│ IPv4 Header:                                             │
│   Src IP:    192.168.0.100                               │
│   Dst IP:    192.168.0.104                               │
│   Protocol:  1 (ICMP)                                    │
│   TTL:       64                                          │
├──────────────────────────────────────────────────────────┤
│ ICMP:                                                    │
│   Type:      8 (Echo Request)                            │
│   ID:        0x1234                                      │
│   Seq:       1                                           │
└──────────────────────────────────────────────────────────┘
```

---

## **PHASE 4: ZIPGATEWAY - READ FROM TAP0**

### **[ZIPGATEWAY] Hàm 1: tapdev_poll() - ĐỌC TỪ TAP0**

**File:** `contiki/cpu/native/net/tapdev6.c:88`
**Interface:** Z/IP Gateway process reading from `tap0`

```c
u16_t tapdev_poll(void)
{
  fd_set fdset;
  struct timeval tv = {0, 0};
  int ret;

  FD_ZERO(&fdset);
  FD_SET(net_fd, &fdset);  // net_fd = tap0 file descriptor

  ret = select(net_fd + 1, &fdset, NULL, NULL, &tv);
  if(ret == 0) return 0;

  // ═══════════════════════════════════════════════════════
  // ĐỌC FRAME TỪ TAP0 VÀO uip_buf
  // ═══════════════════════════════════════════════════════
  ret = read(net_fd, uip_buf, UIP_BUFSIZE);  // ← READ!
  
  return ret;  // 98 bytes (14 Eth + 20 IPv4 + 8 ICMP + 56 data)
}
```

**Buffer sau khi đọc: `uip_buf[]`**
```
Offset  Content
------  -------------------------------------------------------
0-5:    AA:BB:CC:DD:EE:FF  (Dest MAC)
6-11:   11:22:33:44:55:66  (Src MAC)
12-13:  0x08 0x00          (EtherType = IPv4)
14-33:  IPv4 Header (Src: 192.168.0.100, Dst: 192.168.0.104)
34-41:  ICMP Header (Type 8, Echo Request)
42-97:  ICMP Data
```

---

### **[ZIPGATEWAY] Hàm 2: pollhandler() - PHÂN LOẠI PACKET**

**File:** `contiki/cpu/native/net/tapdev-drv.c:71`
**Interface:** Z/IP Gateway packet classifier

```c
static void pollhandler(void)
{
  uip_len = tapdev_poll();  // uip_len = 98
  
  if(uip_len > 0) {
    
    // ═══════════════════════════════════════════════════════
    // KIỂM TRA ETHERTYPE
    // BUF->type = uip_buf[12-13] = 0x0800
    // ═══════════════════════════════════════════════════════
    if(BUF->type == uip_htons(UIP_ETHTYPE_IP)) {  // TRUE!
      
      // ═══════════════════════════════════════════════════════
      // ĐÂY LÀ IPv4 PACKET
      // GỌI IPv4 TO IPv6 NAT
      // ═══════════════════════════════════════════════════════
      ipv46nat_interface_input();  // → Hàm 3
    }

    // Sau NAT, kiểm tra lại EtherType
    if(BUF->type == uip_htons(UIP_ETHTYPE_IPV6)) {
      UIP_IP_BUF->flow = FLOW_FROM_LAN;
      tcpip_input();  // → Hàm 6
    }
  }
}
```

---

### **[ZIPGATEWAY] Hàm 3: ipv46nat_interface_input() - QUYẾT ĐỊNH NAT**

**File:** `src/ipv46_if_handler.c:307`
**Interface:** IPv4/IPv6 NAT module

```c
void ipv46nat_interface_input() 
{
  ip4_hdr_t* ip4h = (ip4_hdr_t*) &(uip_buf[14]);
  nodeid_t node;

  // ═══════════════════════════════════════════════════════
  // KIỂM TRA PROTOCOL
  // ip4h->proto = 1 (ICMP)
  // ═══════════════════════════════════════════════════════
  if(ip4h->proto == UIP_PROTO_TCP) {
    return;  // Skip
  }
  
  else if(ip4h->proto == UIP_PROTO_UDP) {
    // Skip DNS, NTP, DHCP ports
    return;
  }
  
  // ICMP không bị skip!
  
  // ═══════════════════════════════════════════════════════
  // TRA CỨU NAT TABLE
  // 192.168.0.104 → node ID?
  // ═══════════════════════════════════════════════════════
  node = ipv46nat_get_nat_addr((uip_ipv4addr_t*)&ip4h->destipaddr);
  // → Hàm 4
  
  if(node) {
    // ═══════════════════════════════════════════════════════
    // TÌM THẤY NODE! 
    // 192.168.0.104 → node 5 (giả sử)
    // THỰC HIỆN TRANSLATION
    // ═══════════════════════════════════════════════════════
    do_46_translation(node);  // → Hàm 5
  }
}
```

---

### **[ZIPGATEWAY] Hàm 4: ipv46nat_get_nat_addr() - TRA CỨU NODE**

**File:** `src/ipv46_nat.c:36`
**Interface:** NAT table lookup

```c
nodeid_t ipv46nat_get_nat_addr(uip_ipv4addr_t *ip) 
{
  uint16_t i;
  
  // ip->u8 = {192, 168, 0, 104}
  // uip_hostaddr = {192, 168, 0, 1} (Gateway)
  // uip_netmask = {255, 255, 255, 0}
  
  if(uip_ipaddr_maskcmp(ip, &uip_hostaddr, &uip_netmask)) {
    
    // ═══════════════════════════════════════════════════════
    // DUYỆT NAT TABLE
    // ═══════════════════════════════════════════════════════
    for(i = 0; i < nat_table_size; i++) {
      
      // ip_suffix = ip & ~netmask
      // 192.168.0.104 → suffix = 0x6800 (104 in network order)
      
      if(nat_table[i].ip_suffix == (ip->u16[1] & (~uip_netmask.u16[1]))) {
        return nat_table[i].nodeid;  // → node 5
      }
    }
  }
  return 0;
}
```

**NAT Table:**
```
Index  Node ID  IP Suffix    IPv4 Address      IPv6 Address
-----  -------  -----------  ----------------  ------------------
  0      1      0x0100       192.168.0.1       fd00:bbbb::1
  1      5      0x6800       192.168.0.104     fd00:bbbb::5
  2      7      0x6900       192.168.0.105     fd00:bbbb::7
```

**Kết quả:** `node = 5`

---

### **[ZIPGATEWAY] Hàm 5: do_46_translation() - IPv4 → IPv6**

**File:** `src/ipv46_if_handler.c:118`
**Interface:** IPv4 to IPv6 translation engine

```c
static void do_46_translation(nodeid_t node)  // node = 5
{
  ip6_hdr_t* ip6h, __ip6h;
  ip4_hdr_t* ip4h = (ip4_hdr_t*) &(uip_buf[14]);
  struct uip_eth_hdr* ethh = (struct uip_eth_hdr*) uip_buf;
  struct uip_icmp_hdr *icmph;
  uint16_t len, i;
  u8_t *p, *q;

  ip6h = &__ip6h;

  // ═══════════════════════════════════════════════════════
  // TẠO IPv6 HEADER
  // ═══════════════════════════════════════════════════════
  
  // Length: IPv4 total - IPv4 header
  len = UIP_HTONS(ip4h->len) - ((ip4h->vhl & 0xF) << 2);
  // len = 84 - 20 = 64 bytes
  ip6h->len = UIP_HTONS(len);
  
  ip6h->ttl = ip4h->ttl;     // 64
  ip6h->proto = ip4h->proto; // 1 (ICMP)
  ip6h->vtc = 0x60;          // IPv6 version 6
  ip6h->tcflow = 0x00;
  ip6h->flow = 0x00;

  // ═══════════════════════════════════════════════════════
  // DESTINATION IPv6: node 5 → fd00:bbbb::5
  // ═══════════════════════════════════════════════════════
  ipOfNode(&ip6h->destipaddr, node);
  // Result: fd00:bbbb:0000:0000:0000:0000:0000:0005

  // ═══════════════════════════════════════════════════════
  // SOURCE IPv6: 192.168.0.100 → ::ffff:192.168.0.100
  // (IPv4-mapped IPv6)
  // ═══════════════════════════════════════════════════════
  ip4to6_addr(&ip6h->srcipaddr, &ip4h->srcipaddr);
  // Result: 0000:0000:0000:0000:0000:ffff:c0a8:0064

  // ═══════════════════════════════════════════════════════
  // COPY PAYLOAD BACKWARDS (IPv6 header > IPv4 header)
  // ═══════════════════════════════════════════════════════
  p = (u8_t*)ip4h + UIP_HTONS(ip4h->len);
  q = &uip_buf[14 + sizeof(ip6_hdr_t) + len];
  
  uip_len += sizeof(ip6_hdr_t) - sizeof(ip4_hdr_t);  // +20 bytes
  // uip_len = 98 + 20 = 118 bytes
  
  ethh->type = UIP_HTONS(UIP_ETHTYPE_IPV6);  // ← ĐỔI ETHERTYPE!

  // Backward copy
  p--;
  q--;
  for(i = len; i > 0; i--) {
    *q-- = *p--;
  }

  // ═══════════════════════════════════════════════════════
  // GHI IPv6 HEADER
  // ═══════════════════════════════════════════════════════
  memcpy(ip4h, ip6h, sizeof(ip6_hdr_t));
  ip6h = (ip6_hdr_t*)ip4h;

  // ═══════════════════════════════════════════════════════
  // TRANSLATE ICMP TYPE: ICMP → ICMPv6
  // ═══════════════════════════════════════════════════════
  icmph = (struct uip_icmp_hdr*)((u8_t*)ip6h + sizeof(ip6_hdr_t));
  ip6h->proto = UIP_PROTO_ICMP6;  // Protocol = 58 (ICMPv6)
  
  // RFC 2765: ICMPv4 type 8 → ICMPv6 type 128
  if(icmph->type == 8)       // Echo Request
    icmph->type = 128;
  else if(icmph->type == 0)  // Echo Reply
    icmph->type = 129;
  
  // Recalculate checksum
  icmph->icmpchksum = 0;
  icmph->icmpchksum = ~(uip_icmp6chksum());
}
```

**Buffer SAU TRANSLATION: `uip_buf[]`**
```
┌──────────────────────────────────────────────────────────┐
│ AFTER IPv4 → IPv6 TRANSLATION                            │
└──────────────────────────────────────────────────────────┘

Offset  Content
------  -------------------------------------------------------
0-5:    AA:BB:CC:DD:EE:FF  (Dest MAC - không đổi)
6-11:   11:22:33:44:55:66  (Src MAC - không đổi)
12-13:  0x86 0xDD          (EtherType = IPv6) ← ĐÃ ĐỔI!
14-53:  IPv6 Header (40 bytes)
        - Version: 6
        - Next Header: 58 (ICMPv6)
        - Src IP: ::ffff:192.168.0.100  ← IPv4-mapped
        - Dst IP: fd00:bbbb::5           ← Z-Wave node
        - Hop Limit: 64
54-61:  ICMPv6 Header (8 bytes)
        - Type: 128 (Echo Request)       ← Đã đổi từ 8
        - Code: 0
        - Checksum: (recalculated)
        - ID: 0x1234
        - Seq: 1
62-117: ICMP Data (không đổi)

Total: 118 bytes (tăng 20 bytes do IPv6 header lớn hơn)
```

---

### **[ZIPGATEWAY] Hàm 6: tcpip_input() - POST EVENT**

**File:** `contiki/core/net/tcpip.c:584`
**Interface:** Contiki TCP/IP stack entry

```c
void tcpip_input(void)
{
  // ═══════════════════════════════════════════════════════
  // POST EVENT TO TCP/IP PROCESS
  // ═══════════════════════════════════════════════════════
  process_post_synch(&tcpip_process, PACKET_INPUT, NULL);
  
  uip_len = 0;
  uip_ext_len = 0;
}
```

---

### **[ZIPGATEWAY] Hàm 7: uip_input() - IPv6 PROCESSING**

**File:** `contiki/core/net/uip6.c:1000+`
**Interface:** Core IPv6 stack

```c
void uip_input(void)
{
  // Parse IPv6 header
  // Check destination: fd00:bbbb::5
  
  // ═══════════════════════════════════════════════════════
  // KHÔNG PHẢI ĐỊA CHỈ GATEWAY
  // Cần forward đến Z-Wave node
  // ═══════════════════════════════════════════════════════
  
  if(!uip_ds6_is_my_addr(&UIP_IP_BUF->destipaddr)) {
    // Not for us - need forwarding
  }
  
  // ═══════════════════════════════════════════════════════
  // PARSE NEXT HEADER: ICMPv6
  // ═══════════════════════════════════════════════════════
  switch(UIP_IP_BUF->proto) {
    case UIP_PROTO_ICMP6:
      goto icmp6_input;  // → Hàm 8
  }
}
```

---

### **[ZIPGATEWAY] Hàm 8: uip_icmp6_input() - ICMP PROCESSING**

**File:** `contiki/core/net/uip-icmp6.c:200`
**Interface:** ICMPv6 handler

```c
void uip_icmp6_input(void)
{
  UIP_IP_BUF->ttl--;  // Decrement hop limit (forwarding)

  // ═══════════════════════════════════════════════════════
  // KIỂM TRA ICMP TYPE
  // ═══════════════════════════════════════════════════════
  switch(UIP_ICMP_BUF->type) {
    case ICMP6_ECHO_REQUEST:  // Type 128
      // ═══════════════════════════════════════════════════
      // PING REQUEST ĐẾN Z-WAVE NODE
      // Gateway không reply, để node reply
      // Packet sẽ được forward
      // ═══════════════════════════════════════════════════
      goto forward;
  }

forward:
  // Không xử lý local, forward tiếp
  // Packet sẽ đi qua routing
}
```

---

### **[ZIPGATEWAY] Hàm 9: tcpip_ipv6_output() - ROUTING**

**File:** `contiki/core/net/tcpip.c:607`
**Interface:** IPv6 output routing

```c
void tcpip_ipv6_output(void)
{
  uip_ipaddr_t* nexthop;
  uip_ds6_nbr_t* nbr;

  // ═══════════════════════════════════════════════════════
  // ROUTE LOOKUP: fd00:bbbb::5
  // ═══════════════════════════════════════════════════════
  
  if(uip_ds6_is_addr_onlink(&UIP_IP_BUF->destipaddr)) {
    // fd00:bbbb::5 is onlink (in PAN prefix)
    nexthop = &UIP_IP_BUF->destipaddr;
    goto nexthop_done;
  }

nexthop_done:
  // ═══════════════════════════════════════════════════════
  // NEIGHBOR DISCOVERY: IPv6 → L2 Address
  // ═══════════════════════════════════════════════════════
  nbr = uip_ds6_nbr_lookup(nexthop);
  
  if(nbr) {
    // ═══════════════════════════════════════════════════
    // TÌM THẤY NEIGHBOR CACHE
    // ═══════════════════════════════════════════════════
    uip_lladdr_t ll;
    memcpy(&ll, &nbr->lladdr, sizeof(uip_lladdr_t));
    // ll.addr = {00, 00, 00, 00, 00, 05} ← Derived from node ID
    
    // ═══════════════════════════════════════════════════
    // CALL OUTPUT FUNCTION
    // ═══════════════════════════════════════════════════
    tcpip_output(&ll);  // → Hàm 10
  }
}
```

**Neighbor Cache:**
```
IPv6 Address              L2 Address (MAC)     State
-----------------------  -------------------  --------
fd00:bbbb::1             00:00:00:00:00:01    REACHABLE  (Gateway)
fd00:bbbb::5             00:00:00:00:00:05    REACHABLE  (Node 5)
fd00:bbbb::7             00:00:00:00:00:07    REACHABLE  (Node 7)
```

---

### **[ZIPGATEWAY] Hàm 10: zwave_send() - INTERFACE SELECTION**

**File:** `src/ZIP_Router.c:636`
**Interface:** Z-Wave routing decision

```c
static u8_t zwave_send(uip_lladdr_t *addr)
{
  nodeid_t nodeid = 0;
  
  if (addr) {
    // addr->addr = {00, 00, 00, 00, 00, 05}
    
    // ═══════════════════════════════════════════════════════
    // KIỂM TRA: L2 ADDRESS THUỘC PAN?
    // ═══════════════════════════════════════════════════════
    if (memcmp(addr->addr, pan_lladdr.addr, 4) == 0) {
      // ═══════════════════════════════════════════════════
      // ĐÂY LÀ Z-WAVE PAN ADDRESS!
      // ═══════════════════════════════════════════════════
      
      // Lấy node ID từ IPv6
      nodeid = nodeOfIP(&UIP_IP_BUF->destipaddr);
      // nodeid = 5
      
      // ═══════════════════════════════════════════════════
      // QUEUE PACKET CHO Z-WAVE TX
      // ═══════════════════════════════════════════════════
      node_input_queued(nodeid, FALSE);  // → Hàm 11
      
      uip_len = 0;
    }
  }
  return 0;
}
```

---

### **[ZIPGATEWAY] Hàm 11: node_input_queued() - QUEUE PACKET**

**File:** `src/node_queue.c:410`
**Interface:** Z-Wave transmission queue

```c
void node_input_queued(nodeid_t node, BOOL bQueuePriority)
{
  // node = 5
  
  // ═══════════════════════════════════════════════════════
  // THÊM PACKET VÀO QUEUE
  // ═══════════════════════════════════════════════════════
  struct uip_packetqueue_packet *p;
  
  p = uip_packetqueue_alloc(&node_queues[node], 
                            uip_buf,           // IPv6/ICMPv6 packet
                            uip_len,           // 118 bytes
                            CLOCK_SECOND * 60);
  
  // ═══════════════════════════════════════════════════════
  // TRIGGER PROCESSING
  // ═══════════════════════════════════════════════════════
  process_post(&zip_process, ZIP_EVENT_QUEUE_UPDATED, (void*)(intptr_t)node);
}
```

**Queue Entry:**
```
┌──────────────────────────────────────────────────────────┐
│ PACKET IN QUEUE FOR NODE 5                              │
└──────────────────────────────────────────────────────────┘

queue_buf[0-117]:  IPv6/ICMPv6 packet
                   - Src: ::ffff:192.168.0.100
                   - Dst: fd00:bbbb::5
                   - ICMPv6 Echo Request (Type 128)
queue_buf_len: 118
lifetime: 60 seconds
```

---

### **[ZIPGATEWAY] Hàm 12: send_queued_to_node() - DEQUEUE**

**File:** `src/node_queue.c:475`
**Interface:** Queue processor

```c
void send_queued_to_node(nodeid_t node)
{
  struct uip_packetqueue_packet *q;
  
  // ═══════════════════════════════════════════════════════
  // LẤY PACKET TỪ QUEUE
  // ═══════════════════════════════════════════════════════
  q = uip_packetqueue_first(&node_queues[node]);
  
  if (!q) return;
  
  // ═══════════════════════════════════════════════════════
  // COPY VÀO uip_buf
  // ═══════════════════════════════════════════════════════
  uip_len = uip_packetqueue_buflen(q);
  memcpy(uip_buf, uip_packetqueue_buf(q), uip_len);
  
  // ═══════════════════════════════════════════════════════
  // GỬI QUA Z-WAVE
  // ═══════════════════════════════════════════════════════
  if (!ClassicZIPNode_input(node, queue_send_done, FALSE, FALSE)) {
    // → Hàm 13
  }
}
```

---

### **[ZIPGATEWAY] Hàm 13: ClassicZIPNode_input() - IPv6 → Z-Wave**

**File:** `src/ClassicZIPNode.c:920`
**Interface:** IPv6 to Z-Wave conversion

```c
BOOL ClassicZIPNode_input(nodeid_t node, 
                         VOID_CALLBACKFUNC(completedFunc)(BYTE,BYTE*,uint16_t),
                         BOOL bFromMailbox, BOOL already_requeued)
{
  // node = 5
  
  DBG_PRINTF("ClassicZIPNode_input: uip_len:%d \n", uip_len);
  
  // ═══════════════════════════════════════════════════════
  // PARSE IPv6 PACKET
  // ═══════════════════════════════════════════════════════
  struct uip_ip6_hdr *ip6h = (struct uip_ip6_hdr *)&uip_buf[UIP_LLH_LEN];
  
  // ═══════════════════════════════════════════════════════
  // ICMP PACKET → ENCAPSULATE IN ZIP
  // ═══════════════════════════════════════════════════════
  if(ip6h->proto == UIP_PROTO_ICMP6) {
    // Create ZIP packet containing ICMPv6
    // For ping → Will become Z-Wave NOP command
  }
  
  // ═══════════════════════════════════════════════════════
  // SEND TO Z-WAVE
  // ═══════════════════════════════════════════════════════
  return ZW_SendDataZIP_Bridge(node, payload, payload_len, 
                               TRANSMIT_OPTION_ACK | TRANSMIT_OPTION_AUTO_ROUTE,
                               completedFunc);
  // → Hàm 14
}
```

---

## **PHASE 5: Z-WAVE RF TRANSMISSION**

### **[ZIPGATEWAY] Hàm 14: ZW_SendDataAppl() - SERIAL API**

**File:** Serialapi.c
**Interface:** Serial API to Z-Wave chip

```c
BOOL ZW_SendDataAppl(ts_param_t *p, const void *pData, WORD dataLength,
                     ZW_SendDataAppl_Callback_t cbFunc, void* user)
{
  BYTE cmd_buf[300];
  int i = 0;
  
  // ═══════════════════════════════════════════════════════
  // TẠO SERIAL API FRAME
  // ═══════════════════════════════════════════════════════
  cmd_buf[i++] = REQUEST;                    // 0x00
  cmd_buf[i++] = FUNC_ID_ZW_SEND_DATA;       // 0x13
  cmd_buf[i++] = 5;                          // Dest node
  cmd_buf[i++] = dataLength;                 // Data length
  
  // Copy payload (ZIP packet containing ICMPv6)
  memcpy(&cmd_buf[i], pData, dataLength);
  i += dataLength;
  
  cmd_buf[i++] = TRANSMIT_OPTION_ACK | TRANSMIT_OPTION_AUTO_ROUTE;
  cmd_buf[i++] = callback_id;
  
  // ═══════════════════════════════════════════════════════
  // GỬI QUA SERIAL PORT
  // ═══════════════════════════════════════════════════════
  SerialAPI_Queue_Send(cmd_buf, i);  // → Hàm 15
  
  return TRUE;
}
```

**Serial Frame:**
```
┌──────────────────────────────────────────────────────────┐
│ SERIAL API FRAME TO Z-WAVE CHIP                         │
└──────────────────────────────────────────────────────────┘

Byte   Value    Description
-----  -------  --------------------------------------------
0      0x01     SOF (Start of Frame)
1      0x??     Length
2      0x00     REQUEST
3      0x13     FUNC_ID_ZW_SEND_DATA
4      0x05     Destination Node ID = 5
5      0x??     Data Length
6-...  ...      ZIP Packet (containing ICMPv6 Echo Request)
       0x03     TX Options (ACK + AUTO_ROUTE)
       0x??     Callback ID
       0x??     Checksum
```

---

### **[SERIAL PORT] Hàm 15: serial_line_write() - PHYSICAL TX**

**File:** `contiki/platform/linux/dev/serial-line.c`
**Interface:** Serial port (/dev/ttyACM0)

```c
int serial_line_write(const uint8_t *buf, int len)
{
  int ret;
  
  // ═══════════════════════════════════════════════════════
  // GHI RA SERIAL PORT
  // ═══════════════════════════════════════════════════════
  ret = write(serial_fd, buf, len);  // serial_fd = /dev/ttyACM0
  
  if(ret == -1) {
    perror("serial write error");
  }
  
  return ret;
}
```

**Interface:** `/dev/ttyACM0` (USB serial to Z-Wave chip)

---

### **[Z-WAVE CHIP] Processing & RF Transmission**

**Interface:** Z-Wave 700 series chip (internal firmware)

```
┌─────────────────────────────────────────────────────────┐
│ Z-WAVE CHIP INTERNAL PROCESSING                         │
└─────────────────────────────────────────────────────────┘

1. Parse Serial API frame
2. Extract destination node ID = 5
3. Extract payload (ZIP packet)
4. Route lookup (direct or multi-hop)
5. Security encryption (if configured)
6. Build Z-Wave RF frame
7. Modulate and transmit at 868MHz/908MHz
```

**Z-Wave RF Frame:**
```
┌──────────────────────────────────────────────────────────┐
│ Z-WAVE RF FRAME (Over-the-air)                          │
└──────────────────────────────────────────────────────────┘

Field              Value
-----------------  -------------------------------------------
Home ID            0x12345678 (4 bytes)
Source Node        1 (Gateway)
Dest Node          5
Command Class      COMMAND_CLASS_ZIP (0x23)
Command            COMMAND_ZIP_PACKET (0x02)
Payload            IPv6 packet (ICMPv6 Echo Request)
                   - Src: ::ffff:192.168.0.100
                   - Dst: fd00:bbbb::5
                   - ICMPv6 Type 128
Security           Encrypted (if S2 enabled)
CRC                Checksum
```

---

## **PHASE 6: Z-WAVE NODE RECEPTION**

### **[Z-WAVE NODE 5] Receive & Process**

**Interface:** Z-Wave node device (192.168.0.104)

```
┌─────────────────────────────────────────────────────────┐
│ NODE 5 (192.168.0.104 / fd00:bbbb::5)                  │
└─────────────────────────────────────────────────────────┘

1. Receive RF frame at 868MHz/908MHz
2. Demodulate
3. Verify Home ID and Node ID
4. Decrypt (if encrypted)
5. Extract ZIP packet
6. Parse IPv6 header
   - Dst: fd00:bbbb::5 ← This is me!
7. Parse ICMPv6
   - Type 128 (Echo Request)
8. Generate ICMPv6 Echo Reply (Type 129)
9. Send reply back via Z-Wave RF
```

---

## **TÓM TẮT HÀNH TRÌNH:**

```
┌──────────────────────────────────────────────────────────────────┐
│ PING: 192.168.0.100 → 192.168.0.104 (Z-Wave Node 5)             │
└──────────────────────────────────────────────────────────────────┘

[PC]
  │ ICMP Echo Request (Type 8)
  │ Src: 192.168.0.100, Dst: 192.168.0.104
  ↓
[eth0] ← Physical Ethernet
  │ Frame with Dest MAC = Gateway
  ↓
[br-lan] ← Linux Bridge
  │ FDB lookup → forward to tap0
  │ (Frame unchanged)
  ↓
[tap0] ← Virtual TAP device
  │ Frame available in kernel buffer
  ↓
[Z/IP Gateway - tapdev_poll()]
  │ read(net_fd) → uip_buf
  ↓
[pollhandler()]
  │ Check EtherType = 0x0800 (IPv4)
  ↓
[ipv46nat_interface_input()]
  │ Protocol = ICMP
  ↓
[ipv46nat_get_nat_addr()]
  │ 192.168.0.104 → Node 5
  ↓
[do_46_translation(5)]
  │ IPv4 → IPv6 Translation
  │ 192.168.0.100 → ::ffff:192.168.0.100
  │ 192.168.0.104 → fd00:bbbb::5
  │ ICMP Type 8 → ICMPv6 Type 128
  │ EtherType 0x0800 → 0x86DD
  ↓
[tcpip_input()]
  │ Post event to tcpip_process
  ↓
[uip_input()]
  │ IPv6 processing
  │ Dst: fd00:bbbb::5 (not gateway)
  ↓
[uip_icmp6_input()]
  │ ICMPv6 Echo Request
  │ Forward to node
  ↓
[tcpip_ipv6_output()]
  │ Route lookup → onlink (PAN)
  │ ND lookup → L2: 00:00:00:00:00:05
  ↓
[zwave_send()]
  │ L2 addr belongs to PAN
  │ nodeID = 5
  ↓
[node_input_queued()]
  │ Add to node 5's queue
  ↓
[send_queued_to_node()]
  │ Dequeue packet
  ↓
[ClassicZIPNode_input()]
  │ IPv6 → Z-Wave conversion
  ↓
[ZW_SendDataAppl()]
  │ Create Serial API frame
  │ Dest Node = 5
  ↓
[serial_line_write()]
  │ write(/dev/ttyACM0)
  ↓
[Z-Wave Chip]
  │ Parse, route, encrypt
  │ Build RF frame
  ↓
[868MHz/908MHz RF] ← Wireless transmission
  │ Z-Wave protocol
  ↓
[Node 5]
  │ Receive, decrypt, parse
  │ ICMPv6 Echo Request
  │ Generate Echo Reply
  │ Send back via RF
```

---

## **PACKET TRANSFORMATION SUMMARY:**

| Stage | Interface | Src IP | Dst IP | Protocol | Type | Size |
|-------|-----------|--------|--------|----------|------|------|
| eth0 | Physical | 192.168.0.100 | 192.168.0.104 | IPv4/ICMP | Type 8 | 98B |
| br-lan | Bridge | (same) | (same) | (same) | (same) | 98B |
| tap0 | Virtual | (same) | (same) | (same) | (same) | 98B |
| After NAT | Gateway | ::ffff:192.168.0.100 | fd00:bbbb::5 | IPv6/ICMPv6 | Type 128 | 118B |
| Serial | /dev/ttyACM0 | (encapsulated in ZIP) | Node 5 | Serial API | SEND_DATA | ~150B |
| RF | 868MHz | Gateway | Node 5 | Z-Wave | ZIP_PACKET | ~180B |
