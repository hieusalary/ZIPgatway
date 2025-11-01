
Dưới đây là 3 thay đổi nhỏ (một header + một source process, và một chỗ gọi khởi động process) để in log khi Contiki nhận và xuất packet. Chèn vào workspace của bạn như chỉ dẫn, build lại và chạy gateway — log sẽ in ra stderr (hoặc journal nếu service redirect).

1) Tạo header process
````c
// ...existing code...
#ifndef CONTIKI_TRACE_H
#define CONTIKI_TRACE_H

#include "contiki.h"

PROCESS_NAME(contiki_trace_process);

#endif /* CONTIKI_TRACE_H */
// ...existing code...
````

2) Tạo source process (in chi tiết eth / IPv4 / ICMP / IPv6)
````c
// ...existing code...
#include "contiki.h"
#include "contiki-net.h" /* provides tcpip_event, uip_buf/uip_len */
#include <stdio.h>
#include <arpa/inet.h>

PROCESS(contiki_trace_process, "Contiki packet trace process");
AUTOSTART_PROCESSES(&contiki_trace_process); /* optional if you don't start manually */

static void print_eth_ipv4_icmp(const uint8_t *b, unsigned len)
{
  if(len <= 14) return;
  uint16_t eth = (b[12]<<8) | b[13];
  fprintf(stderr, "[TRACE] TAP frame len=%u eth=0x%04x dst=%02x:%02x:%02x:%02x:%02x:%02x src=%02x:%02x:%02x:%02x:%02x:%02x\n",
          len, eth,
          b[0],b[1],b[2],b[3],b[4],b[5],
          b[6],b[7],b[8],b[9],b[10],b[11]);
  if(eth == 0x0800 && len >= 14 + 20) { /* IPv4 */
    const uint8_t *ip = b + 14;
    fprintf(stderr, "  [TRACE] IPv4 proto=%u src=%u.%u.%u.%u dst=%u.%u.%u.%u\n",
            ip[9],
            ip[12],ip[13],ip[14],ip[15],
            ip[16],ip[17],ip[18],ip[19]);
    if(ip[9] == 1 && len >= 14 + 20 + 8) {
      const uint8_t *icmp = b + 14 + 20;
      fprintf(stderr, "    [TRACE] ICMP type=%u code=%u\n", icmp[0], icmp[1]);
    }
  } else if(eth == 0x86DD && len >= 14 + 40) { /* IPv6 */
    char src6[INET6_ADDRSTRLEN], dst6[INET6_ADDRSTRLEN];
    inet_ntop(AF_INET6, b + 14 + 8, src6, sizeof(src6));
    inet_ntop(AF_INET6, b + 14 + 24, dst6, sizeof(dst6));
    fprintf(stderr, "  [TRACE] IPv6 src=%s dst=%s\n", src6, dst6);
  }
}

PROCESS_THREAD(contiki_trace_process, ev, data)
{
  PROCESS_BEGIN();

  fprintf(stderr, "[TRACE] contiki_trace_process started\n");

  while(1) {
    /* Wait until tcpip_event posted (packet arrived from tapdev) */
    PROCESS_WAIT_EVENT_UNTIL(ev == tcpip_event);

    if(uip_len > 0) {
      /* uip_buf is provided by Contiki (raw frame) */
      print_eth_ipv4_icmp((const uint8_t *)uip_buf, (unsigned)uip_len);
      /* Also log that tcpip handler is about to run (tcpip_ipv4_input will be invoked) */
      fprintf(stderr, "[TRACE] tcpip_event received: uip_len=%u\n", (unsigned)uip_len);
    } else {
      fprintf(stderr, "[TRACE] tcpip_event received but uip_len==0\n");
    }

    /* Let regular tcpip processing continue; this process simply observes */
  }

  PROCESS_END();
}
// ...existing code...
````

3) Khởi động process khi Contiki khởi (chèn 1 dòng vào contiki platform main)
- Mở file: contiki-main.c và tìm nơi khởi tạo/ start các process; thêm include và start như sau:
````c
// ...existing code...
#include "contiki/tools/contiki_trace.h" /* add near other includes */
// ...existing code...
int main(int argc, char **argv)
{
  // ...existing code initialisation ...
  /* start the trace process (after Contiki subsystems initialised) */
  process_start(&contiki_trace_process, NULL);
  // ...existing code run loop ...
}
// ...existing code...
````

Nếu bạn không muốn edit contiki-main.c để khởi động tự động, bạn có thể remove AUTOSTART_PROCESSES in contiki_trace.c and call process_start(...) manually in the same contiki-main.c as above.

Cách build & test nhanh
- From repo root:
  - mkdir build && cd build
  - cmake .. && make -j$(nproc)
- Run gateway foreground to see stderr logs (or check journal if run as systemd):
  - Run binary (example path ./zipgateway) as root so it can open /dev/net/tun:
    - sudo ./zipgateway
  - Or if systemd service used:
    - sudo systemctl stop zipgateway
    - Run manually with sudo /path/to/zipgateway &> /tmp/zipgateway.log
    - tail -f /tmp/zipgateway.log
- Khi chạy, gửi một ping từ LAN -> a host behind gateway, hoặc capture with tcpdump on tap0:
  - sudo tcpdump -i tap0 -n icmp
  - Khi ping packet đi vào, bạn sẽ thấy sequence of logs:
    - [TRACE] TAP frame ... (from tapdev read)
    - [TRACE] tcpip_event received: uip_len=...
    - contiki original tcpip logs and later when stack outputs you will get TAP TX logs if you enabled previous tapdev send logging.

Gợi ý: trước khi build, bạn có thể chỉ thử compile the new file quickly by adding it to contiki CMakeLists.txt or ensure contiki build picks up files under contiki/tools; if build does not pick it up, add contiki/tools/contiki_trace.c to CMakeLists.txt SOURCES list or include via your application CMakeLists.

Muốn tôi:
- (A) tạo git patch (git apply) cho 3 file thay đổi, hoặc
- (B) tạo thêm tapdev read/send printf (3 chỗ) để có log end‑to‑end?
