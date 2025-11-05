# Contiki Process & Event Listing by Struct

## 1. `struct process` - All Registered Processes

### Struct Definition
```c
// contiki/core/sys/process.h:316
struct process {
  struct process *next;              // Linked list pointer
  const char *name;                  // Process name string
  PT_THREAD((* thread)(struct pt *, process_event_t, process_data_t));
  struct pt pt;                      // Protothread context
  unsigned char state, needspoll;    // State & poll flag
};
```

### Global Variable
```c
// contiki/core/sys/process.c:57
extern struct process *process_list;  // Head of linked list
```

### All Process Instances (Declared with PROCESS() macro)

#### Core Contiki Processes
```c
// contiki/core/sys/etimer.c:59
PROCESS(etimer_process, "Event timer");

// contiki/core/sys/ctimer.c:93
PROCESS(ctimer_process, "Ctimer process");

// contiki/core/net/tcpip.c:202
PROCESS(tcpip_process, "TCP/IP stack");

// contiki/core/net/tcpip_ipv4.c:209
PROCESS(tcpip_ipv4_process, "TCP/IP IPV4 stack");

// contiki/core/net/resolv.c:171 (or 88 depending on IPv6 config)
PROCESS(resolv_process, "DNS resolver");

// contiki/core/dev/serial-line.c:55
PROCESS(serial_line_process, "Serial driver");
```

#### Driver Processes
```c
// contiki/cpu/native/net/tapdev-drv.c:52
PROCESS(tapdev_process, "TAP driver");

// contiki/cpu/native/net/wpcap-drv.c:43
PROCESS(wpcap_process, "WinPcap driver");

// src/serial_api_process.c:19
PROCESS(serial_api_process, "ZW Serial api");
```

#### ZIP Gateway Application Processes
```c
// src/ZIP_Router.c:101
PROCESS(zip_process, "ZIP process");

// src/ZW_udp_server.c:154
PROCESS(udp_server_process, "UDP server process");

// src/DTLS_server.c:24
PROCESS(dtls_server_process, "DTLS server process");

// src/mDNSService.c:127
PROCESS(mDNS_server_process, "mDNS server process");

// src/ZW_tcp_client.c:87
PROCESS(zip_tcp_client_process, "ZIP TCP client process");

// src/dhcpc2.c:62
PROCESS(dhcp_client_process, "DHCP client process");

// src/transport/ZW_SendDataAppl.c:71
PROCESS(ZW_SendDataAppl_process, "ZW_SendDataAppl_process process");
```

### Process List Structure at Runtime
```
process_list → [etimer_process] → [tcpip_process] → [tapdev_process] → [zip_process] → ... → NULL
                     ↓                  ↓                  ↓                  ↓
                   .name             .name              .name              .name
              "Event timer"     "TCP/IP stack"     "TAP driver"       "ZIP process"
```

---

## 2. `struct event_data` - Event Queue

### Struct Definition
```c
// contiki/core/sys/process.c:65
struct event_data {
  process_event_t ev;      // Event type (unsigned char)
  process_data_t data;     // Event data (void*)
  struct process *p;       // Target process
};
```

### Global Variables
```c
// contiki/core/sys/process.c:72
static struct event_data events[PROCESS_CONF_NUMEVENTS];  // Queue (32 slots default)
static process_num_events_t nevents;  // Current number of events
static process_num_events_t fevent;   // Front index (next event to process)
```

### All Event Types

#### Built-in Events (process.h)
```c
// contiki/core/sys/process.h
#define PROCESS_EVENT_NONE            0x80  // No event
#define PROCESS_EVENT_INIT            0x81  // Process init
#define PROCESS_EVENT_POLL            0x82  // Poll request
#define PROCESS_EVENT_EXIT            0x83  // Process exit
#define PROCESS_EVENT_SERVICE_REMOVED 0x84  // Service removed
#define PROCESS_EVENT_CONTINUE        0x85  // Continue
#define PROCESS_EVENT_MSG             0x86  // Message
#define PROCESS_EVENT_EXITED          0x87  // Process exited
#define PROCESS_EVENT_TIMER           0x88  // Timer expired (most common!)
```

#### Custom Events (Allocated dynamically)

**TCP/IP Stack Events:**
```c
// contiki/core/net/tcpip.c:97-99
process_event_t tcpip_event = 0;          // Allocated at runtime
process_event_t tcpip_icmp6_event = 0;    // Allocated at runtime

// Allocated in tcpip_process init:
tcpip_event = process_alloc_event();
tcpip_icmp6_event = process_alloc_event();
```

**IPv4 Stack Events:**
```c
// contiki/core/net/tcpip_ipv4.c
process_event_t tcpip_ipv4_event = 0;     // Allocated at runtime
```

**DNS Resolver Events:**
```c
// contiki/core/net/resolv.c:86 or 169
process_event_t resolv_event_found = 0;   // Allocated at runtime

// Allocated in resolv_process init:
resolv_event_found = process_alloc_event();
```

**Serial Line Events:**
```c
// contiki/core/dev/serial-line.c:57
process_event_t serial_line_event_message; // Allocated at runtime

// Allocated in serial_line_process init:
serial_line_event_message = process_alloc_event();
```

**Internal TCP/IP Events (not allocated, used directly):**
```c
PACKET_INPUT   // Internal to tcpip_process
TCP_POLL       // Internal to tcpip_process
UDP_POLL       // Internal to tcpip_process
TCP_READY      // Broadcast when TCP stack ready
TUNNEL_READY   // ZIP Gateway specific
```

### Event Queue Example at Runtime
```
events[0] = { ev: PROCESS_EVENT_TIMER,  data: &etimer1,     p: &zip_process }
events[1] = { ev: tcpip_event,          data: &server_conn, p: &udp_server_process }
events[2] = { ev: PROCESS_EVENT_TIMER,  data: &etimer2,     p: &dhcp_client_process }
events[3] = { ev: resolv_event_found,   data: "hostname",   p: PROCESS_BROADCAST }
...
events[31] = { ev: 0, data: NULL, p: NULL }  // Empty slot

fevent = 0   (next event to process is events[0])
nevents = 4  (4 events currently queued)
```

---

## 3. Runtime Mapping

### How to Iterate All Processes
```c
#include "sys/process.h"

void list_all_processes(void) {
    struct process *p;
    int count = 0;
    
    printf("All registered processes:\n");
    for (p = process_list; p != NULL; p = p->next) {
        count++;
        printf("%d. %s (state=%d, needspoll=%d)\n", 
               count, 
               PROCESS_NAME_STRING(p),
               p->state,
               p->needspoll);
    }
    printf("Total: %d processes\n", count);
}
```

### How to Iterate Event Queue (Requires Access to Internal Variables)
```c
// NOTE: events[], nevents, fevent are static in process.c
// You need to expose them or use the dumper tool provided

extern struct event_data events[];
extern process_num_events_t nevents, fevent;

void list_all_events(void) {
    int i;
    printf("Events in queue: %d\n", nevents);
    
    for (i = 0; i < nevents; i++) {
        int idx = (fevent + i) % PROCESS_CONF_NUMEVENTS;
        printf("Event %d: type=%d, target=%s, data=%p\n",
               i,
               events[idx].ev,
               events[idx].p ? PROCESS_NAME_STRING(events[idx].p) : "BROADCAST",
               events[idx].data);
    }
}
```

---

## 4. Complete Process-Event Matrix

| Process | Receives Events | Posts Events |
|---------|-----------------|--------------|
| `etimer_process` | (none) | PROCESS_EVENT_TIMER → all processes with etimers |
| `ctimer_process` | PROCESS_EVENT_TIMER | (executes callbacks directly) |
| `tcpip_process` | PACKET_INPUT, TCP_POLL, UDP_POLL, PROCESS_EVENT_TIMER | tcpip_event → UDP/TCP apps, TCP_READY → BROADCAST |
| `tcpip_ipv4_process` | Similar to tcpip_process | tcpip_ipv4_event → IPv4 apps |
| `resolv_process` | tcpip_ipv4_event, PROCESS_EVENT_TIMER | resolv_event_found → BROADCAST |
| `serial_line_process` | (input from serial) | serial_line_event_message → BROADCAST |
| `tapdev_process` | PROCESS_EVENT_TIMER | PACKET_INPUT → tcpip_process |
| `serial_api_process` | (Z-Wave serial) | Z-Wave events → zip_process |
| `zip_process` | tcpip_event, serial_line_event_message, resolv_event_found, PROCESS_EVENT_TIMER | TUNNEL_READY → BROADCAST |
| `udp_server_process` | tcpip_event, TUNNEL_READY, PROCESS_EVENT_TIMER | (forwards to Z-Wave) |
| `dtls_server_process` | tcpip_event, PROCESS_EVENT_TIMER | (forwards to udp_server) |
| `mDNS_server_process` | tcpip_event, PROCESS_EVENT_TIMER | (responds to mDNS queries) |
| `zip_tcp_client_process` | tcpip_event, tcpip_ipv4_event, resolv_event_found, PROCESS_EVENT_TIMER | (portal communication) |
| `dhcp_client_process` | tcpip_ipv4_event, PROCESS_EVENT_TIMER | (DHCP state machine) |
| `ZW_SendDataAppl_process` | (Z-Wave TX requests), PROCESS_EVENT_TIMER | (Z-Wave TX callbacks) |

---

## 5. Using the Dumper Tool

### Add to your debug code:
```c
#include "sys/process_event_dumper.h"

// In your debug function or main loop:
debug_dump_all();
```

### Example Output:
```
╔════════════════════════════════════════════════════════════════╗
║          CONTIKI PROCESS & EVENT RUNTIME DUMP                  ║
╚════════════════════════════════════════════════════════════════╝

========== PROCESS STATISTICS ==========
Total registered processes:  15
Currently running:           12
Processes needing poll:      3
Current process:             ZIP process
Events in queue:             5 / 32
========================================

========== ALL REGISTERED PROCESSES ==========
No.  Process Name                   State        NeedsPoll
------------------------------------------------------------
1    Event timer                    RUNNING      NO
2    TCP/IP stack                   RUNNING      NO
3    TAP driver                     RUNNING      YES
4    ZIP process                    RUNNING      NO
5    UDP server process             RUNNING      NO
6    DTLS server process            RUNNING      NO
7    mDNS server process            RUNNING      NO
8    ZW Serial api                  RUNNING      NO
9    DHCP client process            RUNNING      NO
10   DNS resolver                   RUNNING      NO
...
------------------------------------------------------------
Total processes: 15

========== EVENT QUEUE ==========
Number of events in queue: 5
Front position (fevent): 0

Pos  Target Process                 Event Type      Event Data
--------------------------------------------------------------------------------
0    ZIP process                    TIMER           0x7ffc1234
1    UDP server process             CUSTOM          0x600abc00
2    DHCP client process            TIMER           0x7ffc5678
3    BROADCAST                      CUSTOM          0x0
4    TCP/IP stack                   POLL            0x0
--------------------------------------------------------------------------------
```

---

## Summary

**2 key structs:**
1. **`struct process`** → Linked list via `process_list`, 15-19 instances
2. **`struct event_data`** → Circular queue `events[32]`, managed by `nevents`/`fevent`

**Access patterns:**
- Process iteration: `for (p = process_list; p != NULL; p = p->next)`
- Event iteration: `for (i = 0; i < nevents; i++) { events[(fevent + i) % 32] }`

**Tools provided:**
- `process_event_dumper.c/h` → Runtime inspection functions
- Use `debug_dump_all()` to see live state
