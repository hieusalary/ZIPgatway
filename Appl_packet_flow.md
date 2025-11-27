## App-layer TX Flow: từ ứng dụng đến sát Serial API

Tài liệu này mô tả chi tiết cách ZIP Gateway xử lý một gói tin ở tầng ứng dụng trước khi ghi xuống Serial API (NCP). Bao gồm sơ đồ gói tin (packet) với đầy đủ trường theo từng nhánh encapsulation: Multi Channel (endpoint), CRC16, Security S0, Security S2. Kèm các đoạn code then chốt đã được trích và giải thích rõ.

### Phạm vi
- Chỉ tập trung tầng app (application layer) cho đường gửi (TX) đến trước khi gọi các API Serial (`ZW_SendData`, `ZW_SendData_Bridge`, `ZW_TransportService_SendData`).
- Không mô tả chi tiết lớp Serial API/wire framing; chỉ nêu điểm gọi ở cuối flow.


## Điểm vào: ZW_SendDataAppl

Ứng dụng gọi:

```c
uint8_t ZW_SendDataAppl(ts_param_t* p,
                        const void *pData,
                        uint16_t dataLength,
                        ZW_SendDataAppl_Callback_t callback,
                        void* user);
```

Diễn biến chính:
- Tạo session và frame buffer: `zw_frame_buffer_create(p, pData, dataLength)`.
- Đưa session vào hàng đợi ứng dụng `session_list`.
- Phát sự kiện `SEND_EVENT_SEND_NEXT` để xử lý tiếp.

Ở bước kế tiếp, phần tử đầu hàng đợi sẽ được gửi qua `send_endpoint(...)` để thực hiện encapsulation ở tầng app.


## Encapsulation app-layer: send_endpoint

Hàm chịu trách nhiệm thêm các lớp encapsulation ở tầng ứng dụng (endpoint/CRC16/S0/S2) trước khi đẩy xuống hàng đợi gửi cấp thấp.

```c
u8_t send_endpoint(ts_param_t* p,
                   const u8_t* data,
                   u16_t len,
                   ZW_SendDataAppl_Callback_t cb,
                   void* user)
{
  u16_t new_len = len;
  static u8_t new_buf[UIP_BUFSIZE];

  // 1) Endpoint encapsulation nếu có
  if (p->dendpoint || p->sendpoint) {
    new_buf[0] = COMMAND_CLASS_MULTI_CHANNEL_V2;      // 0x60
    new_buf[1] = MULTI_CHANNEL_CMD_ENCAP_V2;          // 0x0D
    new_buf[2] = p->sendpoint;                        // nguồn
    new_buf[3] = p->dendpoint;                        // đích
    new_len += 4;
    memcpy(&new_buf[4], data, len);
  } else {
    memcpy(new_buf, data, len);
  }

  // 2) Chọn scheme app-layer (NO/CRC16/S0/S2)
  security_scheme_t scheme = ZW_SendData_scheme_select(p, data, len);

  switch (scheme) {
    case USE_CRC16: {
      // CRC16 Encap nếu đủ điều kiện
      if (new_len < META_DATA_MAX_DATA_SIZE
          && new_buf[0] != COMMAND_CLASS_TRANSPORT_SERVICE
          && new_buf[0] != COMMAND_CLASS_SECURITY
          && new_buf[0] != COMMAND_CLASS_SECURITY_2
          && new_buf[0] != COMMAND_CLASS_CRC_16_ENCAP) {
        WORD crc;
        memmove(new_buf + 2, new_buf, new_len);
        new_buf[0] = COMMAND_CLASS_CRC_16_ENCAP;      // 0x56
        new_buf[1] = CRC_16_ENCAP;                    // 0x01
        crc = zgw_crc16(CRC_INIT_VALUE, (BYTE*) new_buf, new_len + 2);
        new_buf[2 + new_len]     = (crc >> 8) & 0xFF; // CRC High
        new_buf[2 + new_len + 1] = (crc >> 0) & 0xFF; // CRC Low
        new_len += 4;
      }
      return send_data(p, new_buf, new_len, cb, user);
    }
    case NO_SCHEME:
      return send_data(p, new_buf, new_len, cb, user);
    case SECURITY_SCHEME_0:
      // S0 không hỗ trợ multicast
      return sec0_send_data(p, new_buf, new_len, cb, user);
    case SECURITY_SCHEME_2_ACCESS:
    case SECURITY_SCHEME_2_AUTHENTICATED:
    case SECURITY_SCHEME_2_UNAUTHENTICATED:
      p->scheme = scheme;
      return sec2_send_data(p, new_buf, new_len, cb, user);
    default:
      return FALSE;
  }
}
```

Giải thích:
- Endpoint (Multi Channel) được chèn trước mọi lớp khác: thêm 4 byte header và payload gốc nằm bên trong.
- `ZW_SendData_scheme_select` quyết định dùng NO/CRC16/S0/S2 dựa trên RD flags và yêu cầu app.
- Với CRC16, lớp `COMMAND_CLASS_CRC_16_ENCAP` bao ngoài inner payload (có thể đã là Multi Channel).
- Với S0/S2, lớp bảo mật sẽ bao ngoài payload (có thể đã là Multi Channel). Việc tạo nonce/AAD/enc tag do module S0/S2 đảm nhiệm.


## Quy tắc chọn scheme: ZW_SendData_scheme_select

Hàm lấy mask security từ RD cho node đích và cho chính gateway, sau đó chọn “highest common” hoặc CRC16/NO theo điều kiện.

```c
static security_scheme_t ZW_SendData_scheme_select(const ts_param_t* param,
                                                   const u8_t* data,
                                                   u8_t len)
{
  u8_t dst_scheme_mask = GetCacheEntryFlag(param->dnode);
  u8_t src_scheme_mask = GetCacheEntryFlag(MyNodeID);

  // KNOWN_BAD hoặc khung quá ngắn
  if ((len < 2) || (dst_scheme_mask & NODE_FLAG_KNOWN_BAD)
                || (src_scheme_mask & NODE_FLAG_KNOWN_BAD)) {
    if ((param->scheme == USE_CRC16)
        && SupportsCmdClass(param->dnode, COMMAND_CLASS_CRC_16_ENCAP)) {
      return USE_CRC16;
    } else {
      return NO_SCHEME;
    }
  }

  // Common scheme giữa đích và nguồn
  dst_scheme_mask &= src_scheme_mask;

  switch (param->scheme) {
    case AUTO_SCHEME: {
      security_scheme_t s = highest_scheme(dst_scheme_mask);
      if (s == NO_SCHEME && SupportsCmdClass(param->dnode, COMMAND_CLASS_CRC_16_ENCAP)) {
        return USE_CRC16;
      } else {
        return s;
      }
    }
    case NO_SCHEME:      return NO_SCHEME;
    case USE_CRC16:      return USE_CRC16;
    case SECURITY_SCHEME_2_ACCESS:
      if (dst_scheme_mask & NODE_FLAG_SECURITY2_ACCESS) return SECURITY_SCHEME_2_ACCESS;
      break;
    case SECURITY_SCHEME_2_AUTHENTICATED:
      if (dst_scheme_mask & NODE_FLAG_SECURITY2_AUTHENTICATED) return SECURITY_SCHEME_2_AUTHENTICATED;
      break;
    case SECURITY_SCHEME_2_UNAUTHENTICATED:
      if (dst_scheme_mask & NODE_FLAG_SECURITY2_UNAUTHENTICATED) return SECURITY_SCHEME_2_UNAUTHENTICATED;
      break;
    case SECURITY_SCHEME_0:
      if (dst_scheme_mask & NODE_FLAG_SECURITY0) return SECURITY_SCHEME_0;
      break;
    default: break;
  }
  // Nếu không hỗ trợ, ghi cảnh báo và trả lại scheme yêu cầu
  return param->scheme;
}
```

Giải thích:
- `GetCacheEntryFlag(node)` lấy mask security từ RD (per-node). App chỉ chọn scheme là giao giữa gateway và node đích.
- AUTO_SCHEME: ưu tiên S2 Access > S2 Auth > S2 Unauth > S0 > None; nếu None nhưng node hỗ trợ CRC16 thì dùng CRC16.
- Scheme cố định: chỉ hợp lệ nếu node đích có flag tương ứng, nếu không sẽ bị cảnh báo.


## Packet diagrams (đầy đủ trường) theo từng nhánh

Các sơ đồ dưới đây thể hiện rõ thứ tự byte/trường của gói tin ở tầng app sau encapsulation (tức là “payload” sẽ được đưa xuống API Serial). Nếu sau đó payload vẫn quá dài, lớp Transport Service sẽ phân mảnh ở bước thấp hơn (không mô tả ở đây).

### A) NO_SCHEME (không endpoint)

```
Offset  Field                          Value
0       COMMAND_CLASS_X                1 byte
1       CMD_Y                          1 byte
2..N    Parameters                     k bytes
```

Mô tả: Payload ứng dụng nguyên vẹn, không có lớp bao ngoài.

---

### B) Multi Channel Encapsulation (endpoint)

```
Offset  Field                                  Value
0       COMMAND_CLASS_MULTI_CHANNEL_V2         0x60
1       MULTI_CHANNEL_CMD_ENCAP_V2             0x0D
2       Source Endpoint                        p->sendpoint (1 byte)
3       Destination Endpoint                   p->dendpoint (1 byte)
4..(4+L-1) Inner Payload                       L bytes
           (COMMAND_CLASS_X, CMD_Y, params...)
```

Mô tả: Endpoint encap bao ngoài payload gốc. Các lớp CRC16/S0/S2 (nếu có) sẽ bao ngoài toàn khối này.

---

### C) CRC16 Encapsulation

Điều kiện: inner payload không phải TS/SEC/S2/CRC, và `new_len < META_DATA_MAX_DATA_SIZE`.

```
Offset  Field                                  Value
0       COMMAND_CLASS_CRC_16_ENCAP             0x56
1       CRC_16_ENCAP                           0x01
2..(2+L-1) Inner Payload                       L bytes
           (Có thể là Multi Channel ở bên trong)
(2+L)   CRC High                               1 byte
(3+L)   CRC Low                                1 byte
```

Mô tả: CRC tính trên [Header CRC16 + Inner Payload]. Inner Payload có thể đã là Multi Channel.

---

### D) Security Scheme 0 (S0)

```
Offset  Field                                  Value
0       COMMAND_CLASS_SECURITY                  0x98
1       SECURITY_MESSAGE_ENCAPSULATION         0x81
2..X    S0 Nonce/Sequence/Headers              (định dạng S0)
Y..Z    Encrypted Inner Payload                (AES-CBCMAC theo S0)
...     MAC/Checksum                           (S0)
```

Inner Payload (trước mã hóa):

```
COMMAND_CLASS_X, CMD_Y, params...
// Có thể là header Multi Channel nếu dùng endpoint
```

Mô tả: Module S0 tạo nonce/encap, mã hóa payload bên trong; app chỉ gọi `sec0_send_data(...)`.

---

### E) Security Scheme 2 (S2)

```
Offset  Field                                  Value
0       COMMAND_CLASS_SECURITY_2               0x9F
1       SECURITY_2_MESSAGE_ENCAPSULATION      0x03
2..A    S2 Headers (SPAN/Seq/Nonce)           (theo S2)
...     AAD (không truyền; dùng trong CCM)    (homeID, src/dst, endpoints, flags)
B..C    Encrypted Inner Payload                (AES-CCM)
...     Auth Tag (CCM)                         16 bytes (tuỳ cấu hình)
```

Inner Payload (trước mã hóa):

```
COMMAND_CLASS_X, CMD_Y, params...
// Có thể là header Multi Channel nếu dùng endpoint
```

Mô tả: Module S2 sinh nonce, SPAN và dùng AAD cho CCM; mã hóa payload bên trong. App gọi `sec2_send_data(...)`.


## Đưa xuống Serial API

Sau khi encapsulation app-layer hoàn tất, payload đã sẵn sàng để ghi xuống NCP:
- Unicast từ chính gateway: `ZW_SendData(dnode, frame_data, frame_len, tx_flags, cb)`.
- Bridge (snode ≠ MyNodeID): `ZW_SendData_Bridge(snode, dnode, ...)`.
- Payload quá dài và node hỗ trợ Transport Service: `ZW_TransportService_SendData(...)` (phân mảnh ở lớp TS).

Các quyết định này nằm trong `PROCESS_THREAD(ZW_SendDataAppl_process, ...)` ở case `SEND_EVENT_SEND_NEXT_LL`.


## Ghi chú và edge cases

- AUTO_SCHEME: chọn highest common giữa RD flags của node đích và gateway; nếu không có security nhưng node hỗ trợ CRC16 → dùng CRC16.
- S0 multicast: không hỗ trợ.
- Non-secure multicast: TODO trong mã (chưa triển khai đầy đủ).
- LR NOP: nếu gửi `COMMAND_CLASS_NO_OPERATION` đến LR node, app tự thay bằng `COMMAND_CLASS_NO_OPERATION_LR`.
- Firmware Activation Set: sau ACK thành công, SPAN S2 được reset để tránh cơ chế lọc trùng (đã có workaround trong mã).


## Tham chiếu file/mã nguồn

- `src/transport/ZW_SendDataAppl.c`
  - `ZW_SendDataAppl(...)`: tạo session/hàng đợi app.
  - `send_endpoint(...)`: endpoint/CRC16/S0/S2 encapsulation ở app-layer.
  - `ZW_SendData_scheme_select(...)`: chọn scheme dựa trên RD flags.
  - `send_data(...)`: đẩy xuống hàng đợi low-level.
  - `PROCESS_THREAD(... SEND_EVENT_SEND_NEXT_LL ...)`: chọn API Serial phù hợp (TS/Bridge/SendData).
- Các module bảo mật:
  - `sec0_send_data(...)`: S0 encaps.
  - `sec2_send_data(...)`: S2 encaps; `sec2_send_multicast(...)` nếu cấu hình test.
- RD flags: `GetCacheEntryFlag(node)`.


## Kết luận

Tầng app đóng gói payload theo thứ tự: (1) Multi Channel nếu có endpoint, (2) chọn và áp dụng lớp bảo mật hoặc CRC16, (3) giao payload đã encaps cho tầng dưới để ghi xuống Serial API. Các packet diagrams ở trên thể hiện đầy đủ trường/byte theo từng nhánh để bạn đối chiếu và debug.
