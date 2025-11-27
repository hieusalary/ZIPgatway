# Sơ Đồ Luồng ACK trong ZIP Gateway

## 1. Luồng Nhận Packet và Gửi ACK Response

```mermaid
sequenceDiagram
    participant UDP as UDP Socket
    participant Handler as UDPCommandHandler
    participant Parser as ZIP Packet Parser
    participant Session as udp_session
    participant AppHandler as do_app_handler
    participant ACK as send_udp_ack
    participant TX as ZW_SendData_UDP

    UDP->>Handler: Nhận ZIP Packet từ node
    Handler->>Parser: Parse ZIP Packet header
    
    Note over Parser: Kiểm tra flags0 byte
    Parser->>Parser: flags0 & ZIP_PACKET_FLAGS0_ACK_REQ (0x80)
    
    alt ACK Required (bit 7 = 1)
        Parser->>Session: s->ack_req = TRUE
        Parser->>Session: s->seq_no = pZipPacket->seqNo
        Handler->>AppHandler: Xử lý Z-Wave command
        
        AppHandler->>AppHandler: Kiểm tra s->ack_req
        AppHandler->>ACK: send_udp_ack(&s->conn, RES_ACK)
        
        Note over ACK: Tạo ZIP Packet ACK Response
        ACK->>ACK: flags0 = ZIP_PACKET_FLAGS0_ACK_RES (0x40)
        ACK->>ACK: seqNo = s->seq_no (copy từ request)
        ACK->>TX: ZW_SendData_UDP(ackreq=FALSE)
        TX->>UDP: Gửi ACK response về node
    else No ACK Required (bit 7 = 0)
        Parser->>Session: s->ack_req = FALSE
        Handler->>AppHandler: Xử lý Z-Wave command
        Note over AppHandler: Không gửi ACK
    end
```

## 2. Luồng Gửi Packet với Yêu Cầu ACK

```mermaid
sequenceDiagram
    participant App as Application Layer
    participant Send as ZW_SendDataZIP_Ack
    participant Session as udp_tx_session
    participant TX as ZW_SendData_UDP
    participant UDP as UDP Socket
    participant Timer as ACK Timeout Timer
    participant Callback as tx_ack_timeout_notify

    App->>Send: ZW_SendDataZIP_Ack(data, callback)
    
    Note over Send: Tạo TX session với ACK tracking
    Send->>Session: Allocate udp_tx_session
    Send->>Session: session->state = EXPECTING_ACK
    Send->>Session: session->callback = callback
    Send->>Session: session->timeout_time = now + 5s
    
    Send->>TX: ZW_SendData_UDP(ackreq=TRUE)
    
    Note over TX: Set ACK request flag
    TX->>TX: flags0 |= ZIP_PACKET_FLAGS0_ACK_REQ (0x80)
    TX->>TX: seqNo = session->seq_no
    TX->>UDP: Gửi packet đến node
    
    Send->>Timer: Bắt đầu timeout timer
    
    alt ACK Response Nhận Được
        UDP->>Session: Nhận ACK response
        Session->>Session: flags0 & ZIP_PACKET_FLAGS0_ACK_RES
        Session->>Session: Kiểm tra seqNo khớp
        Session->>Session: state = ACK_RECEIVED
        Session->>Callback: callback(TRANSMIT_OK)
        Session->>Session: Free session
        Timer->>Timer: Cancel timeout
    else Timeout (5 seconds)
        Timer->>Callback: tx_ack_timeout_notify()
        Callback->>Session: state = TIMEOUT
        Callback->>Callback: callback(TRANSMIT_TIMEOUT)
        Callback->>Session: Free session
    end
```

## 3. Sơ Đồ Chi Tiết Xử Lý ACK Response

```mermaid
sequenceDiagram
    participant UDP as UDP Socket
    participant Handler as UDPCommandHandler
    participant Parser as Handle ACK Response
    participant Session as Find TX Session
    participant Queue as tx_queue
    participant Callback as Application Callback

    UDP->>Handler: Nhận ZIP Packet
    Handler->>Parser: Parse header
    
    Parser->>Parser: Kiểm tra flags0 & ZIP_PACKET_FLAGS0_ACK_RES
    
    alt Is ACK Response (0x40)
        Parser->>Parser: Lấy seqNo từ packet
        Parser->>Session: Tìm session theo seqNo
        
        alt Session Found
            Session->>Session: Kiểm tra state == EXPECTING_ACK
            Session->>Session: Kiểm tra dest IP/port khớp
            
            alt Valid ACK
                Session->>Queue: Xóa session khỏi tx_queue
                Session->>Callback: callback(TRANSMIT_OK)
                Session->>Session: Free session memory
                Note over Session: ACK xử lý thành công
            else Invalid ACK
                Note over Session: Bỏ qua ACK không hợp lệ
            end
        else Session Not Found
            Note over Parser: ACK cho session đã timeout/không tồn tại
            Parser->>Parser: Bỏ qua packet
        end
    else Not ACK Response
        Handler->>Handler: Xử lý packet bình thường
    end
```

## 4. Sơ Đồ State Machine của TX Session

```mermaid
stateDiagram-v2
    [*] --> IDLE: Khởi tạo session
    IDLE --> EXPECTING_ACK: ZW_SendDataZIP_Ack() gọi
    
    EXPECTING_ACK --> ACK_RECEIVED: ACK response nhận được
    EXPECTING_ACK --> TIMEOUT: 5 seconds timeout
    EXPECTING_ACK --> NACK_RECEIVED: NACK response
    
    ACK_RECEIVED --> [*]: callback(TRANSMIT_OK), free session
    TIMEOUT --> [*]: callback(TRANSMIT_TIMEOUT), free session
    NACK_RECEIVED --> [*]: callback(TRANSMIT_FAIL), free session
    
    note right of EXPECTING_ACK
        - flags0 = 0x80 (ACK_REQ)
        - timeout_time = now + 5s
        - seqNo tracking
    end note
    
    note right of ACK_RECEIVED
        - flags0 = 0x40 (ACK_RES)
        - seqNo phải khớp
        - IP/port phải khớp
    end note
```

## 5. Sơ Đồ Packet Structure với ACK Flags

```
┌─────────────────────────────────────────────────────────────┐
│                      ZIP Packet Header                       │
├─────────┬─────────┬─────────┬─────────┬─────────────────────┤
│ Command │ flags0  │ flags1  │ seqNo   │   payload...        │
│ (0x02)  │         │         │         │                     │
├─────────┼─────────┼─────────┼─────────┼─────────────────────┤
│ 1 byte  │ 1 byte  │ 1 byte  │ 1 byte  │   n bytes           │
└─────────┴─────────┴─────────┴─────────┴─────────────────────┘
           │
           └──► flags0 bits:
                ┌───┬───┬───┬───┬───┬───┬───┬───┐
                │ 7 │ 6 │ 5 │ 4 │ 3 │ 2 │ 1 │ 0 │
                ├───┼───┼───┼───┼───┼───┼───┼───┤
                │ACK│ACK│HDR│SEC│ L │ M │ B │ R │
                │REQ│RES│EXT│   │ P │ E │ E │   │
                └───┴───┴───┴───┴───┴───┴───┴───┘
                 │   │
                 │   └──► Bit 6 (0x40): ACK Response
                 └──────► Bit 7 (0x80): ACK Request

┌─────────────────────────────────────────────────────────────┐
│                   Các Trường Hợp ACK                         │
├──────────────────┬──────────┬──────────┬───────────────────┤
│ Packet Type      │ flags0   │ seqNo    │ Ý Nghĩa           │
├──────────────────┼──────────┼──────────┼───────────────────┤
│ Request + ACK    │ 0x80     │ seq_X    │ Yêu cầu ACK       │
│ ACK Response     │ 0x40     │ seq_X    │ Trả lời ACK       │
│ Request no ACK   │ 0x00     │ seq_X    │ Không cần ACK     │
│ NACK Response    │ 0xC0     │ seq_X    │ Từ chối (NACK)    │
└──────────────────┴──────────┴──────────┴───────────────────┘
```

## 6. Timeline Diagram - ACK Request/Response

```
Gateway                                                    Node
   │                                                         │
   │  ZIP Packet (flags0=0x80, seqNo=42)                   │
   ├────────────────────────────────────────────────────────>│
   │  [ACK_REQ set, data payload included]                  │
   │                                                         │
   │                                      ┌─────────────────┤
   │                                      │ Process packet  │
   │                                      │ Check flags0    │
   │                                      │ bit 7 = 1?      │
   │                                      └─────────────────┤
   │                                                         │
   │  ZIP Packet (flags0=0x40, seqNo=42)                   │
   │<────────────────────────────────────────────────────────┤
   │  [ACK_RES set, no payload, same seqNo]                 │
   │                                                         │
┌──┴──┐                                                      │
│ OK! │ Callback: TRANSMIT_OK                               │
└─────┘                                                      │
   │                                                         │
   
   
   ═══════════════ Trường Hợp Timeout ═══════════════
   
Gateway                                                    Node
   │                                                         │
   │  ZIP Packet (flags0=0x80, seqNo=99)                   │
   ├────────────────────────────────────────X               │
   │  [Packet bị mất]                                       │
   │                                                         │
   │ ⏱ Timeout timer: 5 seconds                            │
   │ ⏱ ⏱ ⏱ ⏱ ⏱                                              │
   │                                                         │
┌──┴────┐                                                    │
│TIMEOUT│ Callback: TRANSMIT_TIMEOUT                        │
└───────┘                                                    │
   │                                                         │
```

## 7. Code Flow Summary

### RX Path (Nhận packet và gửi ACK):
```
UDPCommandHandler() [ZW_udp_server.c:1070]
    │
    ├─► Parse flags0 byte
    │   └─► if (flags0 & 0x80) → s->ack_req = TRUE
    │
    ├─► Store seqNo vào session
    │   └─► s->seq_no = pZipPacket->seqNo
    │
    └─► do_app_handler() [line 836]
        └─► if (s->ack_req)
            └─► send_udp_ack(&s->conn, RES_ACK)
                ├─► flags0 = 0x40 (ACK_RES)
                ├─► seqNo = s->seq_no (echo back)
                └─► ZW_SendData_UDP(ackreq=FALSE)
```

### TX Path (Gửi packet với ACK request):
```
ZW_SendDataZIP_Ack() [ZW_udp_server.c:634]
    │
    ├─► Allocate udp_tx_session
    │   ├─► session->state = EXPECTING_ACK
    │   ├─► session->callback = callback
    │   └─► session->timeout_time = now + 5000ms
    │
    ├─► ZW_SendData_UDP(ackreq=TRUE)
    │   └─► flags0 |= 0x80 (ACK_REQ)
    │
    └─► Start timeout monitoring
        │
        ├─► ACK received → callback(TRANSMIT_OK)
        └─► Timeout → callback(TRANSMIT_TIMEOUT)
```

## 8. Các Thông Số Quan Trọng

| Parameter | Value | Source File | Mô Tả |
|-----------|-------|-------------|-------|
| ACK_REQ flag | 0x80 | ZW_zip_classcmd.h | Bit 7 của flags0 |
| ACK_RES flag | 0x40 | ZW_zip_classcmd.h | Bit 6 của flags0 |
| NACK flag | 0xC0 | ZW_zip_classcmd.h | Bit 7+6 của flags0 |
| ACK timeout | 5000 ms | ZW_udp_server.c | Thời gian chờ ACK |
| seqNo range | 0-255 | ZW_zip_classcmd.h | 1 byte sequence number |
| Max retries | 0 | - | Không retry tự động |

## 9. Edge Cases và Error Handling

### Case 1: Duplicate ACK
```
Gateway nhận 2 ACK với cùng seqNo
→ ACK đầu tiên được xử lý, free session
→ ACK thứ 2 không tìm thấy session, bị bỏ qua
```

### Case 2: Late ACK (sau timeout)
```
Timeout xảy ra → callback(TRANSMIT_TIMEOUT), free session
ACK đến sau đó → Không tìm thấy session, bị bỏ qua
```

### Case 3: Wrong seqNo
```
ACK với seqNo không khớp
→ Tìm session thất bại
→ ACK bị bỏ qua
```

### Case 4: NACK Response
```
Node gửi NACK (flags0 = 0xC0)
→ callback(TRANSMIT_FAIL)
→ Free session
```

## 10. Sơ Đồ Tổng Quan: Client → Gateway → Node

```mermaid
sequenceDiagram
    participant Client as Client (Controller/App)
    participant Gateway as ZIP Gateway
    participant UDP as UDP Socket
    participant Node as Z-Wave Node

    Note over Client,Gateway: 1) Client chuẩn bị Z/IP command (JSON/TCP/HTTP etc.)\n2) Client có thể yêu cầu ACK ứng dụng (không phải ZIP ACK)
    Client->>Gateway: Send Z/IP request (logical)
    Gateway->>Gateway: Build ZIP packet (Command=0x02, flags0, seqNo, payload)
    Gateway->>UDP: ZW_SendData_UDP(ackreq = [0/1], seqNo = X)
    UDP->>Node: UDP packet (ZIP packet on IP/UDP)

    alt Node trả ACK (ACK_RES)
        Node-->>UDP: ZIP packet (flags0=0x40, seqNo = X)
        UDP-->>Gateway: Deliver ACK packet
        Gateway->>Client: Forward result / invoke callback (TRANSMIT_OK)
    else Node trả NACK (NACK)
        Node-->>UDP: ZIP packet (flags0=0xC0, seqNo = X)
        UDP-->>Gateway: Deliver NACK
        Gateway->>Client: Forward result (TRANSMIT_FAIL)
    else No response (Timeout)
        Note over Gateway: Timeout after 5000 ms → TRANSMIT_TIMEOUT
        Gateway->>Client: Notify timeout
    end
```

## 11. Gói tin (Packet Layout) — trực quan

```mermaid
flowchart TB
    %% Basic header fields laid out in order
    Command["1 byte<br/>Command (0x02)"]
    Flags0["1 byte<br/>flags0"]
    Flags1["1 byte<br/>flags1"]
    SeqNo["1 byte<br/>seqNo"]
    Payload["payload (n bytes)"]

    Command --> Flags0
    Flags0 --> Flags1
    Flags1 --> SeqNo
    SeqNo --> Payload

    subgraph Flags0_bits
        b7["bit7: ACK_REQ (0x80)"]
        b6["bit6: ACK_RES (0x40)"]
        b5["bit5: HDR_EXT"]
        b4["bit4: SEC"]
        b3["bit3: L"]
        b2["bit2: M"]
        b1["bit1: B"]
        b0["bit0: R"]
    end

    Flags0 --- b7
    Flags0 --- b6
```

### Giải thích nhanh

- Trường hợp gửi từ Client: Client gửi lệnh đến Gateway; Gateway đóng gói thành ZIP packet, gán seqNo và cờ ACK nếu cần; gửi qua UDP tới Node.
- Trường hợp nhận ACK: Node trả ACK_RES với cùng seqNo; Gateway match session → callback TRANSMIT_OK.
- Timeout/NACK: Khi không nhận ACK trong 5000 ms → TRANSMIT_TIMEOUT; NACK → TRANSMIT_FAIL.

Nếu bạn muốn, tôi có thể:
- Tạo một ảnh PNG/SVG của các sơ đồ mermaid.
- Thêm diagram riêng cho luồng từ Node → Gateway → Client (nếu cần chi tiết hơn).


