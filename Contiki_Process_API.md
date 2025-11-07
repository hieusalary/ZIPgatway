Tôi sẽ liệt kê tất cả các hàm API quan trọng của Contiki process system và trích dẫn code của chúng. 

Read [](file:///c%3A/vscode_src/zipgateway-main/contiki/core/sys/process.h#1-1), lines 1 to 100

Read [](file:///c%3A/vscode_src/zipgateway-main/contiki/core/sys/process.h#200-200), lines 200 to 350

Read [](file:///c%3A/vscode_src/zipgateway-main/contiki/core/sys/process.h#350-350), lines 350 to 450

Read [](file:///c%3A/vscode_src/zipgateway-main/contiki/core/sys/process.h#450-450), lines 450 to 533

Bây giờ tôi sẽ đọc implementation của các hàm này: 

Read [](file:///c%3A/vscode_src/zipgateway-main/contiki/core/sys/process.c)

Hoàn hảo! Bây giờ tôi có đầy đủ thông tin để liệt kê tất cả các hàm API của Contiki process system.

## **DANH SÁCH CÁC HÀM API CỦA CONTIKI PROCESS SYSTEM**

---

## **1. HÀM KHỞI TẠO VÀ QUẢN LÝ PROCESS**

### **1.1. `process_init()` - Khởi tạo process system**
**File:** `contiki/core/sys/process.c:233-243`

```c
void process_init(void)
{
  lastevent = PROCESS_EVENT_MAX;

  nevents = fevent = 0;
#if PROCESS_CONF_STATS
  process_maxevents = 0;
#endif /* PROCESS_CONF_STATS */

  process_current = process_list = NULL;
}
```

**Chức năng:** 
- Khởi tạo hệ thống process (gọi 1 lần duy nhất trong `main()`)
- Reset event counter
- Xóa process list
- Phải gọi TRƯỚC bất kỳ hàm process nào khác

---

### **1.2. `process_start()` - Khởi động process**
**File:** `contiki/core/sys/process.c:107-131`

```c
void process_start(struct process *p, const char *arg) 
{
  struct process *q;

  /* First make sure that we don't try to start a process that is
     already running. */
  for(q = process_list; q != p && q != NULL; q = q->next);

  /* If we found the process on the process list, we bail out. */
  if(q == p) {
    return;
  }
  
  /* Put on the procs list.*/
  p->next = process_list;
  process_list = p;
  p->state = PROCESS_STATE_RUNNING;
  PT_INIT(&p->pt);

  PRINTF("process: starting '%s'\n", PROCESS_NAME_STRING(p));

  /* Post a synchronous initialization event to the process. */
  process_post_synch(p, PROCESS_EVENT_INIT, (process_data_t)arg);
}
```

**Chức năng:**
- Khởi động một process
- Thêm process vào `process_list`
- Gọi process NGAY LẬP TỨC với `PROCESS_EVENT_INIT` (synchronous)
- Nếu process đã chạy rồi → return (nop)

**Ví dụ:**
```c
process_start(&zip_process, NULL);
process_start(&serial_api_process, cfg.serial_port);
```

---

### **1.3. `process_exit()` - Dừng process**
**File:** `contiki/core/sys/process.c:227-230`

```c
void process_exit(struct process *p) 
{
  exit_process(p, PROCESS_CURRENT());
}
```

**Helper function `exit_process()`:**
**File:** `contiki/core/sys/process.c:133-186`

```c
static void exit_process(struct process *p, struct process *fromprocess)
{
  register struct process *q;
  struct process *old_current = process_current;

  PRINTF("process: exit_process '%s'\n", PROCESS_NAME_STRING(p));

  /* Make sure the process is in the process list before we try to
     exit it. */
  for(q = process_list; q != p && q != NULL; q = q->next);
  if(q == NULL) {
    return;
  }

  if(process_is_running(p)) {
    /* Process was running */
    p->state = PROCESS_STATE_NONE;

    /*
     * Post a synchronous event to all processes to inform them that
     * this process is about to exit. This will allow services to
     * deallocate state associated with this process.
     */
    for(q = process_list; q != NULL; q = q->next) {
      if(p != q) {
        call_process(q, PROCESS_EVENT_EXITED, (process_data_t)p);
      }
    }

    if(p->thread != NULL && p != fromprocess) {
      /* Post the exit event to the process that is about to exit. */
      process_current = p;
      p->thread(&p->pt, PROCESS_EVENT_EXIT, NULL);
    }
  }

  if(p == process_list) {
    process_list = process_list->next;
  } else {
    for(q = process_list; q != NULL; q = q->next) {
      if(q->next == p) {
        q->next = p->next;
        break;
      }
    }
  }

  process_current = old_current;
}
```

**Chức năng:**
- Dừng một process
- Gửi `PROCESS_EVENT_EXITED` đến tất cả processes khác
- Gửi `PROCESS_EVENT_EXIT` đến process đang bị exit
- Xóa khỏi `process_list`

**Ví dụ:**
```c
process_exit(&udp_server_process);
process_exit(&tcpip_process);
```

---

### **1.4. `process_is_running()` - Kiểm tra process có đang chạy**
**File:** `contiki/core/sys/process.c:417-420`

```c
int process_is_running(struct process *p)
{
  return p->state != PROCESS_STATE_NONE;
}
```

**Chức năng:**
- Return non-zero nếu process đang chạy
- Return 0 nếu process đã stop

**Ví dụ:**
```c
if (process_is_running(&zip_process)) {
  // Process đang chạy
}
```

---

## **2. HÀM GỬI VÀ XỬ LÝ EVENTS**

### **2.1. `process_post()` - Gửi event BẤT ĐỒNG BỘ**
**File:** `contiki/core/sys/process.c:360-392`

```c
int process_post(struct process *p, process_event_t ev, process_data_t data)
{
  static process_num_events_t snum;

  if(PROCESS_CURRENT() == NULL) {
    PRINTF("process_post: NULL process posts event %d to process '%s', nevents %d\n",
           ev, PROCESS_NAME_STRING(p), nevents);
  } else {
    PRINTF("process_post: Process '%s' posts event %d to process '%s', nevents %d\n",
           PROCESS_NAME_STRING(PROCESS_CURRENT()), ev,
           p == PROCESS_BROADCAST? "<broadcast>": PROCESS_NAME_STRING(p), nevents);
  }

  if(nevents == PROCESS_CONF_NUMEVENTS) {
#if DEBUG
    if(p == PROCESS_BROADCAST) {
      printf("soft panic: event queue is full when broadcast event %d was posted from %s\n", 
             ev, PROCESS_NAME_STRING(process_current));
    } else {
      printf("soft panic: event queue is full when event %d was posted to %s from %s\n", 
             ev, PROCESS_NAME_STRING(p), PROCESS_NAME_STRING(process_current));
    }
#endif /* DEBUG */
    return PROCESS_ERR_FULL;
  }

  snum = (process_num_events_t)(fevent + nevents) % PROCESS_CONF_NUMEVENTS;
  events[snum].ev = ev;
  events[snum].data = data;
  events[snum].p = p;
  ++nevents;

#if PROCESS_CONF_STATS
  if(nevents > process_maxevents) {
    process_maxevents = nevents;
  }
#endif /* PROCESS_CONF_STATS */

  return PROCESS_ERR_OK;
}
```

**Chức năng:**
- Gửi event vào event queue (circular buffer 32 slots)
- Event sẽ được xử lý SAU trong event loop
- Có thể broadcast đến tất cả processes với `PROCESS_BROADCAST`
- Return `PROCESS_ERR_OK` nếu thành công
- Return `PROCESS_ERR_FULL` nếu queue đầy

**Ví dụ:**
```c
// Gửi đến 1 process cụ thể
process_post(&zip_process, ZIP_EVENT_RESET, 0);

// Broadcast đến tất cả processes
process_post(PROCESS_BROADCAST, PROCESS_EVENT_EXIT, 0);
```

---

### **2.2. `process_post_synch()` - Gửi event ĐỒNG BỘ**
**File:** `contiki/core/sys/process.c:396-402`

```c
void process_post_synch(struct process *p, process_event_t ev, process_data_t data)
{
  struct process *caller = process_current;

  call_process(p, ev, data);
  process_current = caller;
}
```

**Chức năng:**
- Gọi process NGAY LẬP TỨC (không qua event queue)
- Block cho đến khi process xử lý xong event
- Dùng trong `process_start()` để gửi `PROCESS_EVENT_INIT`

**Ví dụ:**
```c
process_post_synch(p, PROCESS_EVENT_INIT, (process_data_t)arg);
```

---

### **2.3. `process_poll()` - Yêu cầu poll process**
**File:** `contiki/core/sys/process.c:405-414`

```c
void process_poll(struct process *p)
{
  if(p != NULL) {
    if(p->state == PROCESS_STATE_RUNNING ||
       p->state == PROCESS_STATE_CALLED) {
      p->needspoll = 1;
      poll_requested = 1;
    }
  }
}
```

**Chức năng:**
- Set cờ `needspoll` cho process
- Process sẽ được gọi với `PROCESS_EVENT_POLL` trong event loop
- Thường được gọi từ interrupt handler

**Ví dụ:**
```c
process_poll(&tapdev_process);  // Request đọc packets từ TAP
```

---

### **2.4. `process_alloc_event()` - Cấp phát event ID**
**File:** `contiki/core/sys/process.c:100-103`

```c
process_event_t process_alloc_event(void)
{
  return lastevent++;
}
```

**Chức năng:**
- Cấp phát một global event ID (>128)
- Dùng để tạo custom events

**Ví dụ:**
```c
process_event_t tcpip_event = process_alloc_event();
process_event_t resolv_event_found = process_alloc_event();
```

---

## **3. HÀM EVENT LOOP**

### **3.1. `process_run()` - Chạy event loop**
**File:** `contiki/core/sys/process.c:328-338`

```c
int process_run(void)
{
  /* Process poll events. */
  if(poll_requested) {
    do_poll();
  }

  /* Process one event from the queue */
  do_event();

  return nevents + poll_requested;
}
```

**Helper function `do_poll()`:**
**File:** `contiki/core/sys/process.c:253-266`

```c
static void do_poll(void)
{
  struct process *p;

  poll_requested = 0;
  /* Call the processes that needs to be polled. */
  for(p = process_list; p != NULL; p = p->next) {
    if(p->needspoll) {
      p->state = PROCESS_STATE_RUNNING;
      p->needspoll = 0;
      call_process(p, PROCESS_EVENT_POLL, NULL);
    }
  }
}
```

**Helper function `do_event()`:**
**File:** `contiki/core/sys/process.c:273-323`

```c
static void do_event(void)
{
  static process_event_t ev;
  static process_data_t data;
  static struct process *receiver;
  static struct process *p;

  /*
   * If there are any events in the queue, take the first one and walk
   * through the list of processes to see if the event should be
   * delivered to any of them.
   */

  if(nevents > 0) {

    /* There are events that we should deliver. */
    ev = events[fevent].ev;
    data = events[fevent].data;
    receiver = events[fevent].p;

    /* Since we have seen the new event, we move pointer upwards
       and decrese the number of events. */
    fevent = (fevent + 1) % PROCESS_CONF_NUMEVENTS;
    --nevents;

    /* If this is a broadcast event, we deliver it to all events */
    if(receiver == PROCESS_BROADCAST) {
      for(p = process_list; p != NULL; p = p->next) {
        /* If we have been requested to poll a process, we do this in
           between processing the broadcast event. */
        if(poll_requested) {
          do_poll();
        }
        call_process(p, ev, data);
      }
    } else {
      /* This is not a broadcast event, so we deliver it to the
         specified process. */
      if(ev == PROCESS_EVENT_INIT) {
        receiver->state = PROCESS_STATE_RUNNING;
      }

      /* Make sure that the process actually is running. */
      call_process(receiver, ev, data);
    }
  }
}
```

**Chức năng:**
- Xử lý poll requests (nếu có)
- Lấy 1 event từ queue và dispatch đến process
- Return số events còn lại trong queue
- Được gọi trong vòng lặp `while(1)` trong `main()`

**Ví dụ trong `main()`:**
```c
while(1) {
  while(process_run()) {
    // Xử lý events cho đến khi queue rỗng
  }
  
  // Đợi I/O events
  select(...);
}
```

---

### **3.2. `process_nevents()` - Đếm số events**
**File:** `contiki/core/sys/process.c:341-344`

```c
int process_nevents(void)
{
  return nevents + poll_requested;
}
```

**Chức năng:**
- Return tổng số events đang chờ xử lý

---

## **4. HELPER FUNCTION (Internal)**

### **4.1. `call_process()` - Gọi process thread**
**File:** `contiki/core/sys/process.c:188-223`

```c
static void call_process(struct process *p, process_event_t ev, process_data_t data)
{
  int ret;

#if DEBUG
  if(p->state == PROCESS_STATE_CALLED) {
    printf("process: process '%s' called again with event %bu\n", 
           PROCESS_NAME_STRING(p), ev);
  }
#endif

  if((p->state & PROCESS_STATE_RUNNING) && p->thread != NULL) {
    PRINTF("process: calling process '%s' with event %bu\n", 
           PROCESS_NAME_STRING(p), ev);
    
    process_current = p;
    p->state = PROCESS_STATE_CALLED;
    
    ret = p->thread(&p->pt, ev, data);
    
    if(ret == PT_EXITED || ret == PT_ENDED || ev == PROCESS_EVENT_EXIT) {
      exit_process(p, p);
    } else {
      p->state = PROCESS_STATE_RUNNING;
    }
  }
}
```

**Chức năng:**
- Gọi thread function của process
- Set `process_current` 
- Xử lý return value (PT_EXITED → exit process)
- Internal function, không gọi trực tiếp

---

## **5. MACROS QUAN TRỌNG**

### **5.1. `PROCESS_CURRENT()` - Lấy process hiện tại**
**File:** `contiki/core/sys/process.h:406`

```c
#define PROCESS_CURRENT() process_current
```

**Ví dụ:**
```c
struct process *current = PROCESS_CURRENT();
```

---

### **5.2. `PROCESS()` - Khai báo process**
**File:** `contiki/core/sys/process.h:304-310`

```c
#define PROCESS(name, strname)              \
  PROCESS_THREAD(name, ev, data);           \
  struct process name = { NULL, strname,    \
                          process_thread_##name }
```

**Ví dụ:**
```c
PROCESS(zip_process, "ZIP Router");
```

Expand thành:
```c
static PT_THREAD(process_thread_zip_process(struct pt *process_pt,
                                             process_event_t ev,
                                             process_data_t data));
struct process zip_process = { NULL, "ZIP Router",
                               process_thread_zip_process };
```

---

### **5.3. `PROCESS_THREAD()` - Định nghĩa process thread**
**File:** `contiki/core/sys/process.h:277-280`

```c
#define PROCESS_THREAD(name, ev, data)                  \
static PT_THREAD(process_thread_##name(struct pt *process_pt, \
                                       process_event_t ev,     \
                                       process_data_t data))
```

**Ví dụ:**
```c
PROCESS_THREAD(zip_process, ev, data)
{
  PROCESS_BEGIN();
  // ... code ...
  PROCESS_END();
}
```

---

## **6. TỔNG KẾT CÁC HÀM API**

| Hàm | Chức năng | Synchronous? |
|-----|-----------|--------------|
| `process_init()` | Khởi tạo process system | N/A |
| `process_start()` | Khởi động process | **Yes** (gọi INIT ngay) |
| `process_exit()` | Dừng process | **Yes** |
| `process_is_running()` | Kiểm tra process đang chạy | N/A |
| `process_post()` | Gửi event bất đồng bộ | **No** (vào queue) |
| `process_post_synch()` | Gửi event đồng bộ | **Yes** (gọi ngay) |
| `process_poll()` | Request poll process | **No** (set flag) |
| `process_alloc_event()` | Cấp phát event ID | N/A |
| `process_run()` | Chạy event loop | N/A |
| `process_nevents()` | Đếm số events | N/A |
| `PROCESS_CURRENT()` | Lấy process hiện tại | N/A |

---

## **7. BUILT-IN EVENTS**

**File:** `contiki/core/sys/process.h:90-98`

```c
#define PROCESS_EVENT_NONE            0x80
#define PROCESS_EVENT_INIT            0x81  // Process được start
#define PROCESS_EVENT_POLL            0x82  // Poll request
#define PROCESS_EVENT_EXIT            0x83  // Process sắp exit
#define PROCESS_EVENT_SERVICE_REMOVED 0x84  // Service bị remove
#define PROCESS_EVENT_CONTINUE        0x85  // Continue sau PAUSE
#define PROCESS_EVENT_MSG             0x86  // Message event
#define PROCESS_EVENT_EXITED          0x87  // Process đã exit
#define PROCESS_EVENT_TIMER           0x88  // Timer expired
```

Similar code found with 1 license type
