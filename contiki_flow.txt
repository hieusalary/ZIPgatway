@startuml
title Contiki Event Flow - Detailed Sequence
skinparam sequenceMessageAlign center

participant "contiki-main" as main
participant "etimer_process" as etimer
participant "tapdev_process" as tapdev
participant "tcpip_process" as tcpip
participant "resolv_process" as resolv
participant "zip_process" as zip
participant "udp_server_process" as udpsrv
participant "serial_api_process" as serial

== INITIALIZATION ==
main -> tapdev : process_start(&tapdev_process)
main -> zip : autostart(&zip_process)
zip -> tcpip : process_start(&tcpip_process)
activate tcpip
tcpip -> tcpip : tcpip_event = process_alloc_event()
tcpip -> tcpip : tcpip_icmp6_event = process_alloc_event()
zip -> udpsrv : process_start(&udp_server_process)
zip -> serial : process_start(&serial_api_process)
zip -> resolv : process_start(&resolv_process)\n(if portal enabled)

== PACKET INGRESS: LAN → ZIP ==
tapdev -> tapdev : tapdev_poll()\nread frame from tap0
tapdev -> tcpip : process_post_synch(&tcpip_process,\nPACKET_INPUT, NULL)
activate tcpip
tcpip -> tcpip : packet_input()\nuip_input() parse packet
tcpip -> tcpip : lookup UDP connection by port
tcpip -> tcpip : tcpip_uipcall()\nget appstate.p
tcpip -> udpsrv : process_post_synch(appstate.p,\ntcpip_event, appstate.state)
deactivate tcpip
activate udpsrv
udpsrv -> udpsrv : PROCESS_YIELD() returns\nev==tcpip_event, data==server_conn
udpsrv -> udpsrv : tcpip_handler()\nparse Z-Wave/IP packet
udpsrv -> zip : forward to application logic
deactivate udpsrv

== PACKET EGRESS: ZIP → LAN ==
zip -> tcpip : uip_send(buf, len)\nprepare response
tcpip -> tcpip : tcpip_ipv6_output()
tcpip -> tapdev : outputfunc=tapdev_send()\nwrite to tap0
tapdev -> tapdev : write() to /dev/net/tun

== TIMER EVENT ==
etimer -> etimer : Timer expires
etimer -> zip : process_post(zip_process,\nPROCESS_EVENT_TIMER, etimer)
activate zip
zip -> zip : PROCESS_YIELD() returns\nev==PROCESS_EVENT_TIMER
zip -> zip : handle timeout\n(e.g., retry, cleanup)
deactivate zip

== DNS RESOLUTION ==
zip -> resolv : resolv_query("portal.example.com")
activate resolv
resolv -> tcpip : UDP send to DNS server
tcpip -> tapdev : tapdev_send() DNS query
... wait for reply ...
tapdev -> tcpip : process_post_synch(PACKET_INPUT)
tcpip -> resolv : process_post_synch(tcpip_event, ...)
resolv -> resolv : parse DNS response
resolv -> zip : process_post(PROCESS_BROADCAST,\nresolv_event_found, hostname)
deactivate resolv
activate zip
zip -> zip : hostname resolved,\nconnect to portal
deactivate zip

== Z-WAVE SERIAL EVENT ==
serial -> serial : select() on serial port\ndata available
serial -> serial : process_post_synch(&serial_line_process,\nserial_line_event_message, data)
serial -> zip : PROCESS_YIELD() returns\nev==serial_line_event_message
activate zip
zip -> zip : ApplicationCommandHandlerZIP()\nhandle Z-Wave command
deactivate zip

== TCP CONNECTION READY ==
tcpip -> tcpip : TCP handshake complete
tcpip -> zip : process_post(PROCESS_BROADCAST,\ntcpip_event, TCP_READY)
activate zip
zip -> zip : TUNNEL_READY state\nsystem_net_hook(up)
zip -> udpsrv : process_post(udp_server_process,\nTUNNEL_READY, NULL)
zip -> serial : Start listening for Z-Wave
deactivate zip

note over main, serial
  Key mechanisms:
  1. process_post(): Async event delivery (queued)
  2. process_post_synch(): Immediate delivery (blocks)
  3. PROCESS_YIELD(): Wait for next event
  4. appstate.p: Maps connections to processes
  5. tcpip_event: Generic "data available" notification
end note

@enduml
