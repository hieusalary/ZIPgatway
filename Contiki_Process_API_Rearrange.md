Tôi sẽ mapping các API với events tương ứng và sắp xếp theo thứ tự logic từ khởi tạo đến kết thúc.

## **MAPPING CÁC API VỚI EVENTS - SẮP XẾP THEO THỨ TỰ LIFECYCLE**

---

## **GIAI ĐOẠN 1: KHỞI TẠO HỆ THỐNG**

### **1.1. `process_init()` → Không có event**
```c
void process_init(void)
{
  lastevent = PROCESS_EVENT_MAX;
  nevents = fevent = 0;
  process_current = process_list = NULL;
}
```

**Event:** Không có
**Thứ tự:** Đầu tiên trong `main()`, trước tất cả
**Gọi từ:** `main()` - line 1 lần duy nhất

---

## **GIAI ĐOẠN 2: KHỞI ĐỘNG PROCESS**

### **2.1. `process_start()` → Tự động gửi `PROCESS_EVENT_INIT`**
```c
void process_start(struct process *p, const char *arg)
{
  // ... thêm vào process_list ...
  p->state = PROCESS_STATE_RUNNING;
  PT_INIT(&p->pt);
  
  // GỬI EVENT INIT ĐỒNG BỘ
  process_post_synch(p, PROCESS_EVENT_INIT, (process_data_t)arg);
}
```

**Event được gửi:** `PROCESS_EVENT_INIT` (0x81)
**Cách gửi:** Synchronous (qua `process_post_synch()`)
**Khi nào:** Process nhận event này NGAY KHI được start
**Thứ tự:** Sau `process_init()`, theo thứ tự start trong code

**Process nhận được:**
```c
PROCESS_THREAD(my_process, ev, data)
{
  PROCESS_BEGIN();  // ← Chỉ chạy khi ev == PROCESS_EVENT_INIT
  
  if(ev == PROCESS_EVENT_INIT) {
    // Khởi tạo process
  }
  
  PROCESS_END();
}
```

**Ví dụ thứ tự trong main:**
```c
process_init();                           // Bước 1
process_start(&etimer_process, 0);        // Bước 2 → INIT
process_start(&tapdev_process, 0);        // Bước 3 → INIT
autostart_start(autostart_processes);     // Bước 4 → INIT cho zip_process
```

---

## **GIAI ĐOẠN 3: CHẠY EVENT LOOP - XỬ LÝ EVENTS**

### **3.1. `process_poll()` → Tạo `PROCESS_EVENT_POLL`**
```c
void process_poll(struct process *p)
{
  if(p != NULL) {
    p->needspoll = 1;
    poll_requested = 1;
  }
}
```

**Event được tạo:** `PROCESS_EVENT_POLL` (0x82)
**Cách gửi:** Không qua queue, set flag `needspoll`
**Khi nào:** 
- Từ interrupt handler
- Khi cần process xử lý ngay (ví dụ: packet đến)
**Xử lý trong:** `process_run()` → `do_poll()` → gọi process với `PROCESS_EVENT_POLL`

**Process nhận được:**
```c
PROCESS_THREAD(tapdev_process, ev, data)
{
  // POLLHANDLER chạy TRƯỚC PROCESS_BEGIN
  PROCESS_POLLHANDLER(pollhandler());  // ← Chạy khi ev == POLL
  
  PROCESS_BEGIN();
  // ...
  process_poll(&tapdev_process);  // Request poll
  // ...
  PROCESS_END();
}
```

**Flow:**
```
Interrupt/Code → process_poll(&p) → Set needspoll flag
                                    ↓
Event loop → process_run() → do_poll() → call_process(p, PROCESS_EVENT_POLL)
                                         ↓
                            Process's POLLHANDLER chạy
```

---

### **3.2. `process_post()` → Gửi BẤT KỲ event nào (async)**
```c
int process_post(struct process *p, process_event_t ev, process_data_t data)
{
  // ... thêm vào events queue ...
  events[snum].ev = ev;
  events[snum].data = data;
  events[snum].p = p;
  ++nevents;
  return PROCESS_ERR_OK;
}
```

**Event được gửi:** 
- Custom events (>128): `tcpip_event`, `resolv_event_found`, `ZIP_EVENT_*`
- Built-in events: `PROCESS_EVENT_TIMER`, `PROCESS_EVENT_CONTINUE`, `PROCESS_EVENT_EXIT`
- Broadcast events: `PROCESS_BROADCAST`

**Cách gửi:** Asynchronous - vào circular buffer (32 slots)
**Khi nào:** Được xử lý SAU trong event loop
**Xử lý trong:** `process_run()` → `do_event()` → lấy từ queue → gọi process

**Ví dụ các events:**

#### **3.2.1. Custom Events**
```c
// Allocate event ID (1 lần duy nhất)
process_event_t tcpip_event = process_alloc_event();  // ID = 129
process_event_t resolv_event_found = process_alloc_event();  // ID = 130
process_event_t ZIP_EVENT_RESET = process_alloc_event();  // ID = 131

// Gửi events
process_post(&zip_process, ZIP_EVENT_RESET, 0);
process_post(&zip_process, tcpip_event, NULL);
process_post(PROCESS_BROADCAST, resolv_event_found, &result);
```

#### **3.2.2. `PROCESS_EVENT_TIMER` (0x88)**
```c
// Từ etimer_process khi timer expire
process_post(p, PROCESS_EVENT_TIMER, &et);
```

**Process nhận:**
```c
if(ev == PROCESS_EVENT_TIMER) {
  if(data == &my_timer) {
    // Timer expired
    etimer_restart(&my_timer);
  }
}
```

#### **3.2.3. `PROCESS_EVENT_CONTINUE` (0x85)**
```c
// Từ PROCESS_PAUSE() macro
#define PROCESS_PAUSE() do {
  process_post(PROCESS_CURRENT(), PROCESS_EVENT_CONTINUE, NULL);
  PROCESS_WAIT_EVENT();
} while(0)
```

**Process nhận:**
```c
PROCESS_PAUSE();  // Yield rồi tiếp tục
// Code tiếp tục sau khi nhận PROCESS_EVENT_CONTINUE
```

#### **3.2.4. `PROCESS_EVENT_EXIT` (0x83)**
```c
// Gửi để yêu cầu process exit
process_post(&zip_process, PROCESS_EVENT_EXIT, 0);
```

**Process nhận:**
```c
PROCESS_EXITHANDLER({
  // Cleanup code
  close_resources();
});

PROCESS_BEGIN();
// ...
if(ev == PROCESS_EVENT_EXIT) {
  // Process sắp exit
  cleanup();
}
PROCESS_END();
```

---

### **3.3. `process_post_synch()` → Gửi BẤT KỲ event nào (sync)**
```c
void process_post_synch(struct process *p, process_event_t ev, process_data_t data)
{
  struct process *caller = process_current;
  call_process(p, ev, data);  // ← GỌI NGAY
  process_current = caller;
}
```

**Event được gửi:** Bất kỳ event nào
**Cách gửi:** Synchronous - gọi NGAY KHÔNG qua queue
**Khi nào:** 
- Trong `process_start()` để gửi `PROCESS_EVENT_INIT`
- Khi cần xử lý đồng bộ

**Ví dụ:**
```c
// Gửi custom event synchronously
process_post_synch(&my_process, my_custom_event, &data);
// ← Process đã xử lý xong event khi đến đây
```

---

### **3.4. `process_run()` → Xử lý tất cả events**
```c
int process_run(void)
{
  /* Process poll events. */
  if(poll_requested) {
    do_poll();  // ← Xử lý PROCESS_EVENT_POLL
  }

  /* Process one event from the queue */
  do_event();  // ← Xử lý events từ queue

  return nevents + poll_requested;
}
```

**Events được xử lý:**
1. **PROCESS_EVENT_POLL** (từ `process_poll()`)
2. **Tất cả events trong queue** (từ `process_post()`)

**Thứ tự ưu tiên:**
1. Poll events (xử lý trước)
2. Events trong queue (FIFO)

**Được gọi trong:** `main()` event loop
```c
while(1) {
  while(process_run()) {
    // Xử lý cho đến khi queue rỗng
  }
  select(...);  // Đợi I/O
}
```

---

## **GIAI ĐOẠN 4: KẾT THÚC PROCESS**

### **4.1. `process_exit()` → Gửi `PROCESS_EVENT_EXIT` & `PROCESS_EVENT_EXITED`**
```c
void process_exit(struct process *p)
{
  exit_process(p, PROCESS_CURRENT());
}

static void exit_process(struct process *p, struct process *fromprocess)
{
  // 4.1.1: Gửi PROCESS_EVENT_EXITED đến TẤT CẢ processes khác
  for(q = process_list; q != NULL; q = q->next) {
    if(p != q) {
      call_process(q, PROCESS_EVENT_EXITED, (process_data_t)p);
    }
  }
  
  // 4.1.2: Gửi PROCESS_EVENT_EXIT đến chính process đó
  if(p->thread != NULL && p != fromprocess) {
    process_current = p;
    p->thread(&p->pt, PROCESS_EVENT_EXIT, NULL);
  }
  
  // 4.1.3: Xóa khỏi process_list
  // ...
}
```

**Events được gửi:**

#### **4.1.1. `PROCESS_EVENT_EXITED` (0x87)**
**Gửi đến:** TẤT CẢ processes khác (trừ process đang exit)
**Cách gửi:** Synchronous (gọi `call_process()` trực tiếp)
**Mục đích:** Thông báo cho các process khác biết process này đã exit

**Các process khác nhận được:**
```c
if(ev == PROCESS_EVENT_EXITED) {
  struct process *exited = (struct process *)data;
  // Cleanup resources liên quan đến process đã exit
  if(exited == &mDNS_server_process) {
    rd_destroy();
    NetworkManagement_mdns_exited();
  }
}
```

#### **4.1.2. `PROCESS_EVENT_EXIT` (0x83)**
**Gửi đến:** Chính process đang exit
**Cách gửi:** Synchronous (gọi thread function trực tiếp)
**Mục đích:** Cho phép process cleanup trước khi exit

**Process đang exit nhận được:**
```c
PROCESS_EXITHANDLER({
  // Cleanup code - chạy TRƯỚC PROCESS_BEGIN
  close_files();
  free_memory();
});

PROCESS_BEGIN();
// ...
if(ev == PROCESS_EVENT_EXIT) {
  // Cleanup - chạy TRONG process thread
  tapdev_exit();
}
PROCESS_END();
```

---

## **BẢNG TỔNG HỢP: API → EVENTS MAPPING**

| API | Event được tạo/gửi | Sync/Async | Recipient | Thứ tự lifecycle |
|-----|-------------------|------------|-----------|------------------|
| `process_init()` | Không có | N/A | N/A | **1. Khởi tạo hệ thống** |
| `process_start()` | **`PROCESS_EVENT_INIT`** (0x81) | **Sync** | Process được start | **2. Khởi động process** |
| `process_poll()` | **`PROCESS_EVENT_POLL`** (0x82) | Async (flag) | Process cụ thể | **3. Runtime - Poll** |
| `process_post()` | **Custom/Built-in events** | **Async** (queue) | Process cụ thể hoặc BROADCAST | **3. Runtime - Events** |
| `process_post_synch()` | **Bất kỳ event** | **Sync** | Process cụ thể | **3. Runtime - Sync call** |
| `process_run()` | Xử lý POLL + Queue events | N/A | Tất cả processes | **3. Runtime - Event loop** |
| `process_exit()` | **`PROCESS_EVENT_EXIT`** (0x83) <br> **`PROCESS_EVENT_EXITED`** (0x87) | **Sync** | Process đang exit <br> Tất cả processes khác | **4. Kết thúc process** |

---

## **TIMELINE: THỨ TỰ EVENTS TRONG LIFECYCLE**

```
═══════════════════════════════════════════════════════════════════════════
GIAI ĐOẠN 1: KHỞI TẠO HỆ THỐNG
═══════════════════════════════════════════════════════════════════════════
main()
  ↓
[1] process_init()
      → Không có events
      → Khởi tạo process_list = NULL, events queue rỗng

═══════════════════════════════════════════════════════════════════════════
GIAI ĐOẠN 2: KHỞI ĐỘNG CÁC PROCESSES
═══════════════════════════════════════════════════════════════════════════

[2] procinit_init()
      ↓
    process_start(&etimer_process, 0)
      → Gửi PROCESS_EVENT_INIT (SYNC) → etimer_process
      → etimer_process nhận INIT và khởi tạo
      ← Return về main()

[3] process_start(&tapdev_process, 0)
      → Gửi PROCESS_EVENT_INIT (SYNC) → tapdev_process
      → tapdev_process nhận INIT:
         - tapdev_init() tạo tap0
         - tcpip_set_outputfunc()
         - process_poll(&tapdev_process)  ← Set needspoll flag
         - etimer_set()
         - PROCESS_YIELD()
      ← Return về main()

[4] autostart_start(autostart_processes)
      ↓
    process_start(&zip_process, NULL)
      → Gửi PROCESS_EVENT_INIT (SYNC) → zip_process
      → zip_process nhận INIT:
         - ZIP_Router_Reset()
           - process_start(&serial_api_process)
               → PROCESS_EVENT_INIT → serial_api_process
           - process_start(&tcpip_process)
               → PROCESS_EVENT_INIT → tcpip_process
           - process_start(&udp_server_process)
               → PROCESS_EVENT_INIT → udp_server_process
           - process_start(&dtls_server_process)
               → PROCESS_EVENT_INIT → dtls_server_process
           - process_start(&mDNS_server_process)
               → PROCESS_EVENT_INIT → mDNS_server_process
           - process_start(&zip_tcp_client_process)
               → PROCESS_EVENT_INIT → zip_tcp_client_process
           - process_start(&resolv_process)
               → PROCESS_EVENT_INIT → resolv_process
           - process_start(&dhcp_client_process)
               → PROCESS_EVENT_INIT → dhcp_client_process
         - PROCESS_YIELD()
      ← Return về main()

═══════════════════════════════════════════════════════════════════════════
GIAI ĐOẠN 3: EVENT LOOP - XỬ LÝ RUNTIME EVENTS
═══════════════════════════════════════════════════════════════════════════

[5] while(1) { process_run(); }
      ↓
    ┌─────────────────────────────────────────────────────────────────┐
    │ ITERATION 1: Xử lý POLL events                                  │
    └─────────────────────────────────────────────────────────────────┘
    process_run()
      ↓
    if(poll_requested) {  ← TRUE (từ tapdev_process)
      do_poll()
        ↓
        for(p = process_list; p != NULL; p = p->next) {
          if(p->needspoll) {  ← tapdev_process có needspoll = 1
            call_process(p, PROCESS_EVENT_POLL, NULL)
              ↓
            tapdev_process nhận PROCESS_EVENT_POLL:
              → PROCESS_POLLHANDLER(pollhandler()) chạy
              → pollhandler():
                 - tapdev_poll() đọc packet từ tap0
                 - Demux theo EtherType
                 - tcpip_input() → process_post(&tcpip_process, tcpip_event)
    }
    
    ┌─────────────────────────────────────────────────────────────────┐
    │ ITERATION 2: Xử lý events từ queue                              │
    └─────────────────────────────────────────────────────────────────┘
    do_event()
      ↓
    if(nevents > 0) {
      ev = events[fevent].ev;      ← tcpip_event (129)
      data = events[fevent].data;
      receiver = events[fevent].p;  ← tcpip_process
      
      call_process(tcpip_process, tcpip_event, data)
        ↓
      tcpip_process nhận tcpip_event:
        → uip_process(UIP_DATA)
        → Dispatch đến application:
           - udp_new() đã lưu appstate.p = &udp_server_process
           - tcpip_uipcall() gọi application callback
           - process_post(&udp_server_process, tcpip_event, conn)
    }
    
    ┌─────────────────────────────────────────────────────────────────┐
    │ ITERATION 3: Xử lý UDP packet đến application                   │
    └─────────────────────────────────────────────────────────────────┘
    do_event()
      ↓
      ev = tcpip_event (129)
      receiver = udp_server_process
      
      call_process(udp_server_process, tcpip_event, conn)
        ↓
      udp_server_process nhận tcpip_event:
        → ZW_Handleframe() xử lý Z-Wave packet
        → Gọi command handlers
        → Gửi reply
    
    ┌─────────────────────────────────────────────────────────────────┐
    │ ITERATION 4: Xử lý TIMER events                                 │
    └─────────────────────────────────────────────────────────────────┘
    do_event()
      ↓
      ev = PROCESS_EVENT_TIMER (0x88)
      receiver = zip_process
      
      call_process(zip_process, PROCESS_EVENT_TIMER, &backup_timer)
        ↓
      zip_process nhận PROCESS_EVENT_TIMER:
        if(ev == PROCESS_EVENT_TIMER) {
          if(data == &backup_timer) {
            // Thực hiện backup
            zgw_backup();
          }
        }
    
    ┌─────────────────────────────────────────────────────────────────┐
    │ ITERATION 5: Xử lý CUSTOM events                                │
    └─────────────────────────────────────────────────────────────────┘
    do_event()
      ↓
      ev = ZIP_EVENT_BRIDGE_INITIALIZED (131)
      receiver = zip_process
      
      call_process(zip_process, ZIP_EVENT_BRIDGE_INITIALIZED, NULL)
        ↓
      zip_process nhận ZIP_EVENT_BRIDGE_INITIALIZED:
        → ApplicationInitProtocols()
        → NetworkManagement_Init()
        → send_nodelist()

═══════════════════════════════════════════════════════════════════════════
GIAI ĐOẠN 4: KẾT THÚC PROCESS
═══════════════════════════════════════════════════════════════════════════

[6] process_exit(&mDNS_server_process)
      ↓
    exit_process(&mDNS_server_process)
      ↓
    ┌─────────────────────────────────────────────────────────────────┐
    │ BƯỚC 1: Thông báo cho TẤT CẢ processes khác                     │
    └─────────────────────────────────────────────────────────────────┘
    for(q = process_list; q != NULL; q = q->next) {
      if(p != q) {
        call_process(q, PROCESS_EVENT_EXITED, &mDNS_server_process)
          ↓
        zip_process nhận PROCESS_EVENT_EXITED:
          if(ev == PROCESS_EVENT_EXITED) {
            if(data == &mDNS_server_process) {
              rd_destroy();
              NetworkManagement_mdns_exited();
            }
          }
          ↓
        tcpip_process nhận PROCESS_EVENT_EXITED:
          // Cleanup routes liên quan mDNS
          ↓
        udp_server_process nhận PROCESS_EVENT_EXITED:
          // Cleanup connections
      }
    }
    
    ┌─────────────────────────────────────────────────────────────────┐
    │ BƯỚC 2: Gửi EXIT event cho chính process đó                     │
    └─────────────────────────────────────────────────────────────────┘
    mDNS_server_process->thread(&pt, PROCESS_EVENT_EXIT, NULL)
      ↓
    mDNS_server_process nhận PROCESS_EVENT_EXIT:
      PROCESS_EXITHANDLER({
        // Cleanup sockets
        close(mdns_socket);
      });
      
      if(ev == PROCESS_EVENT_EXIT) {
        // Final cleanup
        mdns_cleanup();
      }
    
    ┌─────────────────────────────────────────────────────────────────┐
    │ BƯỚC 3: Xóa khỏi process_list                                   │
    └─────────────────────────────────────────────────────────────────┘
    Remove mDNS_server_process from process_list
    ← Return về caller
```

---

## **BẢNG ƯU TIÊN XỬ LÝ EVENTS**

| Priority | Event Type | Source | Khi nào xử lý |
|----------|-----------|--------|---------------|
| **1 (Cao nhất)** | `PROCESS_EVENT_POLL` | `process_poll()` | Ngay trong `do_poll()` (trước queue) |
| **2** | `PROCESS_EVENT_INIT` | `process_start()` | **SYNC** - ngay khi start (không vào queue) |
| **3** | Custom events (>128) | `process_post()` | Async - theo thứ tự FIFO trong queue |
| **4** | `PROCESS_EVENT_TIMER` | etimer_process | Async - theo thứ tự FIFO trong queue |
| **5** | `PROCESS_EVENT_CONTINUE` | `PROCESS_PAUSE()` | Async - theo thứ tự FIFO trong queue |
| **6** | `PROCESS_EVENT_EXIT` | `process_exit()` | **SYNC** - ngay khi exit |
| **7** | `PROCESS_EVENT_EXITED` | `process_exit()` | **SYNC** - broadcast đến tất cả |

---

## **TÓM TẮT: API USAGE PATTERNS**

```c
// ═══════════════════════════════════════════════════════════════
// PATTERN 1: Khởi động process
// ═══════════════════════════════════════════════════════════════
process_init();                    // 1 lần duy nhất
process_start(&my_process, arg);   // → PROCESS_EVENT_INIT (SYNC)

// ═══════════════════════════════════════════════════════════════
// PATTERN 2: Request poll (từ interrupt/driver)
// ═══════════════════════════════════════════════════════════════
process_poll(&tapdev_process);     // → PROCESS_EVENT_POLL (flag)
// Sẽ được xử lý trong process_run() → do_poll()

// ═══════════════════════════════════════════════════════════════
// PATTERN 3: Gửi async event
// ═══════════════════════════════════════════════════════════════
process_post(&zip_process, ZIP_EVENT_RESET, 0);
// → Vào queue, xử lý sau trong do_event()

// ═══════════════════════════════════════════════════════════════
// PATTERN 4: Broadcast event
// ═══════════════════════════════════════════════════════════════
process_post(PROCESS_BROADCAST, PROCESS_EVENT_EXIT, 0);
// → Tất cả processes nhận được

// ═══════════════════════════════════════════════════════════════
// PATTERN 5: Gửi sync event (block đến khi xong)
// ═══════════════════════════════════════════════════════════════
process_post_synch(&my_process, my_event, &data);
// ← Process đã xử lý xong khi return

// ═══════════════════════════════════════════════════════════════
// PATTERN 6: Event loop
// ═══════════════════════════════════════════════════════════════
while(1) {
  while(process_run()) {  // Xử lý POLL + queue events
    // Loop cho đến khi queue rỗng
  }
  select(...);  // Đợi I/O
}

// ═══════════════════════════════════════════════════════════════
// PATTERN 7: Dừng process
// ═══════════════════════════════════════════════════════════════
process_exit(&my_process);
// → PROCESS_EVENT_EXIT (SYNC) đến my_process
// → PROCESS_EVENT_EXITED (SYNC broadcast) đến tất cả processes khác
```
