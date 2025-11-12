# S2 ENCRYPTION FLOW - Theo Dõi Chi Tiết Keys và Encryption

## Tổng quan

Tài liệu này trace **CHI TIẾT** quá trình sử dụng **Network Keys** và **ECDH Keys** để encrypt packets trong S2 Security framework.

---

## 1. Key Storage - Keys được lưu ở đâu?

### **A) Network Keys (Symmetric AES-128 Keys)**

**Location**: NVM (Non-Volatile Memory) trong Z-Wave chip

**File**: `src/transport/s2_keystore.c`

```c
// NVM Layout
typedef struct {
  uint8_t magic[4];                      // ZIPMAGIC
  uint8_t security_netkey[16];          // S0 network key
  uint8_t security2_key[3][16];         // S2 keys (3 classes)
    // [0] = KEY_CLASS_S2_UNAUTHENTICATED
    // [1] = KEY_CLASS_S2_AUTHENTICATED  
    // [2] = KEY_CLASS_S2_ACCESS
  uint8_t security2_lr_key[2][16];      // S2 Long Range keys
    // [0] = KEY_CLASS_S2_AUTHENTICATED_LR
    // [1] = KEY_CLASS_S2_ACCESS_LR
  uint8_t ecdh_priv_key[32];            // ECDH private key
} zip_nvm_config_t;
```

**Đọc keys từ NVM**:
```c
// s2_keystore.c:141
bool keystore_network_key_read(uint8_t keyclass, uint8_t *buf)
{
  uint8_t assigned_keys;
  
  // Check if key is assigned
  nvm_config_get(assigned_keys, &assigned_keys);
  if (0 == (keyclass & assigned_keys)) {
    return 0;  // Key not assigned
  }
  
  switch(keyclass)
  {
    case KEY_CLASS_S0:
      nvm_config_get(security_netkey, buf);  // ← ĐỌC KEY TỪ NVM
      break;
      
    case KEY_CLASS_S2_UNAUTHENTICATED:
      nvm_config_get(security2_key[0], buf);  // ← ĐỌC KEY TỪ NVM
      break;
      
    case KEY_CLASS_S2_AUTHENTICATED:
      nvm_config_get(security2_key[1], buf);  // ← ĐỌC KEY TỪ NVM
      break;
      
    case KEY_CLASS_S2_ACCESS:
      nvm_config_get(security2_key[2], buf);  // ← ĐỌC KEY TỪ NVM
      break;
      
    // Long Range keys...
  }
  
  DBG_PRINTF("Key class 0x%02x: %s\n", keyclass, 
             security_keyclass_name(keyclass));
  print_key(buf);  // Debug print key (if enabled)
  
  return 1;
}
```

**Physical Storage**: 
- **Z-Wave 700 series**: Internal NVM3 flash
- **Z-Wave 500 series**: External EEPROM hoặc Serial API NVM
- **Address offset**: Định nghĩa trong `zip_nvm_config_t`

---

### **B) ECDH Keys (Elliptic Curve Keys)**

**ECDH Private Key** (32 bytes):
```c
// Lưu trong NVM
uint8_t ecdh_priv_key[32];  // 256-bit EC private key

// Access via keystore
bool keystore_private_key_read(uint8_t *buf);
```

**ECDH Public Key** (64 bytes):
```c
// Được tính toán từ private key khi cần
uint8_t ecdh_public_key[64];  // X (32 bytes) + Y (32 bytes)

// Calculation: libs2/crypto/curve25519
void crypto_scalarmult_curve25519(
  uint8_t *public_key,        // Output: 32-byte public key
  const uint8_t *private_key, // Input: 32-byte private key
  const uint8_t *basepoint    // Generator point
);
```

---

## 2. Key Derivation - Keys được sinh ra như thế nào?

### **A) Network Keys Creation (Khi Factory Reset)**

**File**: `src/transport/S2_wrap.c:274`

```c
void sec2_create_new_network_keys() {
  uint8_t net_key[16];
  
  for (int c = 0; c < 6; c++) {  // 6 key classes
    // Generate random 128-bit key
    if (!dev_urandom(sizeof(net_key), net_key)) {
      ERR_PRINTF("Failed to generate network key\n");
      return;
    }
    
    // Write to NVM
    keystore_network_key_write(1 << c, net_key);
      // 1<<0 = KEY_CLASS_S2_UNAUTHENTICATED
      // 1<<1 = KEY_CLASS_S2_AUTHENTICATED
      // 1<<2 = KEY_CLASS_S2_ACCESS
      // 1<<3 = KEY_CLASS_S2_AUTHENTICATED_LR
      // 1<<4 = KEY_CLASS_S2_ACCESS_LR
  }
  
  DBG_PRINTF("New S2 network keys created\n");
}
```

**Random source**: `/dev/urandom` (Linux) hoặc hardware RNG

---

### **B) ECDH Key Creation**

```c
// S2_wrap.c:261
void sec2_create_new_dynamic_ecdh_key()
{
  uint8_t ecdh_dynamic_key[32];
  
  // Generate random using CTR-DRBG (NIST SP 800-90A)
  AES_CTR_DRBG_Generate(&s2_ctr_drbg, ecdh_dynamic_key);      // ← 16 bytes
  AES_CTR_DRBG_Generate(&s2_ctr_drbg, &ecdh_dynamic_key[16]); // ← 16 bytes more
  
  // Store in keystore
  keystore_private_key_write(ecdh_dynamic_key);
}
```

**CTR-DRBG** (Counter mode Deterministic Random Bit Generator):
- NIST-certified PRNG
- Seeded from `/dev/urandom`
- State maintained in `s2_ctr_drbg` global variable

---

### **C) Derived Keys (Nonce Key, Encryption Key)**

**Khi nào derive**: Sau key exchange trong inclusion

**File**: `libs2/protocol/S2.c`

```c
// KDF (Key Derivation Function) per NIST SP 800-108
void kderiv(
  const uint8_t *shared_secret,  // ECDH shared secret (32 bytes)
  const uint8_t *constant_nk,    // Constant for nonce key
  const uint8_t *constant_ek,    // Constant for encryption key  
  uint8_t *nonce_key,            // Output: 16-byte nonce key
  uint8_t *enc_key               // Output: 16-byte encryption key
)
{
  // CMAC-based KDF
  // nonce_key = CMAC(shared_secret, constant_nk)
  // enc_key   = CMAC(shared_secret, constant_ek)
}
```

**Storage của derived keys**:
```c
// S2 context structure
struct S2 {
  struct {
    uint8_t nonce_key[16];   // ← Key để sinh nonce
    uint8_t enc_key[16];     // ← Key để encrypt payload
  } sg[N_SEC_CLASS];  // Array cho mỗi security class
};
```

---

## 3. Encryption Flow - Packet được mã hóa như thế nào?

### **Entry Point 1: Application gửi command**

```c
// ZIP_Router.c:761 - ApplicationCommandHandlerZIP() nhận frame
ZW_APPLICATION_TX_BUFFER *pCmd;

// Route đến ClassicZIPNode
ClassicZIPNode_sendRequest(data, len, callback);
```

### **Entry Point 2: Transport Service**

```c
// ZW_TransportService.c
void ZW_TransportService_SendData(
  ts_param_t* p,        // Transport parameters
  uint8_t* data,        // Plain text data
  uint16_t data_len,
  callback_func cb
)
{
  // Check security scheme
  if (p->scheme >= SECURITY_SCHEME_2_UNAUTHENTICATED) {
    // S2 encryption
    sec2_send_data(p, data, data_len, cb, user);  // ← ENTRY POINT
  } else if (p->scheme == SECURITY_SCHEME_0) {
    // S0 encryption
    sec0_send_data(...);
  } else {
    // No security
    ZW_SendData(...);
  }
}
```

---

### **Step 1: S2 Wrapper - Setup Connection**

**File**: `src/transport/S2_wrap.c:351`

```c
uint8_t sec2_send_data(ts_param_t* p, uint8_t* data, uint16_t len,
                       ZW_SendDataAppl_Callback_t callback, void* user)
{
  s2_connection_t s2_con;
  
  // Setup connection parameters
  s2_con.l_node = p->snode;        // Local node (gateway)
  s2_con.r_node = p->dnode;        // Remote node (destination)
  s2_con.zw_tx_options = p->tx_flags;
  s2_con.tx_options = 0;
  
  // Verify delivery for GET commands or mailbox
  if (p->force_verify_delivery || 
      mb_is_busy() || 
      CommandAnalyzerIsGet(data[0], data[1]))
  {
    s2_con.tx_options |= S2_TXOPTION_VERIFY_DELIVERY;
  }
  
  // Map scheme to class_id
  s2_con.class_id = p->scheme - SECURITY_SCHEME_2_UNAUTHENTICATED;
    // SECURITY_SCHEME_2_UNAUTHENTICATED → class_id = 0
    // SECURITY_SCHEME_2_AUTHENTICATED   → class_id = 1
    // SECURITY_SCHEME_2_ACCESS          → class_id = 2
  
  // Save callback
  s2_send_callback = callback;
  s2_send_user = user;
  
  // Call libs2 encryption engine
  if (S2_send_data(s2_ctx, &s2_con, data, len)) {  // ← CALL LIBS2
    return 1;  // Success
  } else {
    s2_send_callback = cb_save;
    s2_send_user = user_save;
    return 0;  // Failure (S2 busy)
  }
}
```

---

### **Step 2: S2 Library - Check SPAN State**

**File**: `libs2/protocol/S2.c`

```c
uint8_t S2_send_data(
  struct S2* p_context, 
  const s2_connection_t* con,
  const uint8_t* buf, 
  uint16_t len
)
{
  return S2_send_data_all_cast(p_context, con, buf, len, SEND_MSG);
}

static uint8_t S2_send_data_all_cast(
  struct S2* p_context,
  const s2_connection_t* con,
  const uint8_t* buf,
  uint16_t len,
  event_t ev
)
{
  CTX_DEF  // Define ctxt = p_context
  
  // Check if S2 is idle
  if (ctxt->fsm != IDLE) {
    return 0;  // Busy, reject
  }
  
  // Check buffer size
  if (len > S2_MSG_BUFFER_SIZE) {
    return 0;  // Packet too large
  }
  
  // Copy parameters
  ctxt->peer = *con;
  memcpy(ctxt->buf, buf, len);
  ctxt->length = len;
  
  // Check SPAN (Security Pairwise Association Node)
  if (!S2_span_ok(ctxt, con)) {
    // SPAN not synchronized, need to request nonce first
    struct SPAN *span = find_span_by_node(ctxt, con);
    S2_send_nonce_get(ctxt);  // Send NONCE_GET
    ctxt->fsm = WAIT_NONCE_RAPORT;  // Wait for NONCE_REPORT
    ctxt->retry = 2;  // Allow 2 retries
    ctxt->event_send_done = ev;
    return 1;
  }
  
  // SPAN OK, encrypt and send immediately
  S2_encrypt_and_send(ctxt);  // ← ENCRYPTION HAPPENS HERE
  
  ctxt->fsm = SENDING_MSG;
  ctxt->event_send_done = ev;
  ctxt->retry = 2;
  
  return 1;
}
```

**SPAN State Machine**:
```
SPAN_NOT_USED          → Initial state
SPAN_NO_SEQ            → Created but not synchronized
SPAN_SOS_REMOTE_NONCE  → Received remote nonce, ready to sync
SPAN_NEGOTIATED        → Synchronized, can encrypt ✓
```

---

### **Step 3: S2 Encryption Engine - CHÍNH XÁC Ở ĐÂY KEYS ĐƯỢC SỬ DỤNG**

**File**: `libs2/protocol/S2.c:421`

```c
void S2_encrypt_and_send(struct S2* p_context)
{
  CTX_DEF
  uint8_t aad[64];           // Additional Authenticated Data
  uint16_t aad_len;
  uint8_t ei_sender[16];     // Entropy input sender (random)
  uint8_t ei_receiver[16];   // Entropy input receiver (from NONCE_REPORT)
  uint8_t nonce[16];         // 128-bit nonce for this packet
  uint8_t* ciphertext;
  uint8_t* msg;
  uint16_t hdr_len;
  uint16_t shdr_len;
  uint16_t msg_len;
  
  struct SPAN *span = find_span_by_node(ctxt, &ctxt->peer);
  
  // ===== BUILD S2 MESSAGE HEADER =====
  msg = ctxt->workbuf;
  msg[0] = COMMAND_CLASS_SECURITY_2;      // 0x9F
  msg[1] = SECURITY_2_MESSAGE_ENCAPSULATION;  // 0x03
  msg[2] = span->tx_seq;                  // Sequence number
  msg[3] = 0;                             // Flags (filled later)
  
  hdr_len = 4;
  
  // ===== HANDLE SPAN SYNCHRONIZATION (If needed) =====
  if (span->state == SPAN_SOS_REMOTE_NONCE)
  {
    DBG_PRINTF("SPAN_SOS_REMOTE_NONCE. class_id: %u\n", ctxt->peer.class_id);
    
    // Generate random entropy (16 bytes)
    AES_CTR_DRBG_Generate(&s2_ctr_drbg, ei_sender);  // ← USES CTR-DRBG
    
    // Get remote entropy from received NONCE_REPORT
    memcpy(ei_receiver, span->d.r_nonce, sizeof(ei_receiver));
    
    // ===== INSTANTIATE NONCE RNG =====
    // This creates the synchronized nonce generator
    next_nonce_instantiate(
      &span->d.rng,                          // Output: Nonce RNG state
      ei_sender,                             // Input: Our random
      ei_receiver,                           // Input: Their random
      ctxt->sg[ctxt->peer.class_id].nonce_key  // ← NONCE KEY ĐỌC TỪ NVM
    );
    
    span->class_id = ctxt->peer.class_id;
    span->state = SPAN_NEGOTIATED;
    
    // Add SN (Sender Nonce) extension to header
    uint8_t* ext_data = &msg[4];
    *ext_data++ = 2 + 16;  // Extension length
    *ext_data++ = S2_MSG_EXTHDR_CRITICAL_FLAG | S2_MSG_EXTHDR_TYPE_SN;
    memcpy(ext_data, ei_sender, 16);  // Include our entropy
    hdr_len += 2 + 16;
  }
  
  // ===== ADD MULTICAST EXTENSIONS (if needed) =====
  // ... (skipped for brevity)
  
  // ===== PREPARE CIPHERTEXT =====
  ciphertext = &msg[hdr_len];
  
  // Add secure extensions (MPAN, etc)
  shdr_len = S2_add_mpan_extensions(ctxt, ciphertext);
  if (shdr_len) {
    msg[3] |= SECURITY_2_MESSAGE_ENCAPSULATION_PROPERTIES1_ENCRYPTED_EXTENSION_BIT_MASK;
  }
  
  // Copy plaintext payload
  memcpy(ciphertext + shdr_len, ctxt->buf, ctxt->length);
  
  // ===== BUILD AAD (Additional Authenticated Data) =====
  aad_len = S2_make_aad(
    ctxt,
    ctxt->peer.l_node,      // Sender node ID
    ctxt->peer.r_node,      // Receiver node ID
    msg,                    // S2 message
    hdr_len,                // Header length
    ctxt->length + shdr_len + hdr_len + AUTH_TAG_LEN,  // Total length
    aad,                    // Output buffer
    sizeof(aad)
  );
  
  // AAD format:
  // [NodeID_sender][NodeID_receiver][HomeID][Length][S2_Header]
  
  // ===== GENERATE NONCE =====
  next_nonce_generate(&span->d.rng, nonce);  // ← Sinh nonce từ RNG
  
  DBG_PRINTF("Encryption class %i\n", ctxt->peer.class_id);
  DBG_PRINTF("Nonce: "); print_hex(nonce, 16);
  DBG_PRINTF("Key: ");   print_hex(ctxt->sg[ctxt->peer.class_id].enc_key, 16);
  DBG_PRINTF("AAD: ");   print_hex(aad, aad_len);
  
  // ===== AES-CCM ENCRYPTION =====
  // CHÍNH XÁC TẠI ĐÂY: NETWORK KEY ĐƯỢC SỬ DỤNG
  
#if defined(ZWAVE_PSA_SECURE_VAULT) && defined(ZWAVE_PSA_AES)
  // ===== OPTION A: Hardware Security Module (PSA Crypto API) =====
  size_t out_len;
  uint32_t key_id = ZWAVE_CCM_TEMP_ENC_KEY_ID;
  
  // Import key into secure vault (hardware)
  zw_wrap_aes_key_secure_vault(
    &key_id,
    ctxt->sg[ctxt->peer.class_id].enc_key,  // ← ENCRYPTION KEY ĐỌC TỪ NVM
    ZW_PSA_ALG_CCM
  );
  
  // Encrypt using PSA API (hardware accelerated)
  zw_psa_aead_encrypt_ccm(
    key_id,                          // Key handle
    nonce,                           // Nonce (16 bytes)
    ZWAVE_PSA_AES_NONCE_LENGTH,     // = 13
    aad,                             // AAD
    aad_len,
    ciphertext,                      // Plaintext input
    ctxt->length + shdr_len,        // Plaintext length
    ciphertext,                      // Ciphertext output (in-place)
    ctxt->length + shdr_len + ZWAVE_PSA_AES_MAC_LENGTH,
    &out_len
  );
  
  msg_len = out_len;
  
  // Remove key from vault (security)
  zw_psa_destroy_key(key_id);
  
#else
  // ===== OPTION B: Software AES-CCM =====
  msg_len = CCM_encrypt_and_auth(
    ctxt->sg[ctxt->peer.class_id].enc_key,  // ← ENCRYPTION KEY ĐỌC TỪ NVM
    nonce,                                    // Nonce (16 bytes)
    aad,                                      // AAD
    aad_len,
    ciphertext,                               // Plaintext/ciphertext buffer
    ctxt->length + shdr_len                  // Plaintext length
  );
  // Output: ciphertext overwritten with encrypted data + 8-byte MAC
  // msg_len = plaintext_len + 8
#endif
  
  DBG_PRINTF("Ciphertext: "); print_hex(ciphertext, msg_len);
  
  // ===== SEND ENCRYPTED PACKET =====
  S2_send_raw(ctxt, msg, msg_len + hdr_len);
    // Calls callback: send_data_cb() defined in S2_wrap.c
    // Which calls: ZW_SendDataEx() to transmit over RF
  
  span->tx_seq++;  // Increment sequence number for next packet
}
```

---

### **Step 4: AES-CCM Encryption Details**

**File**: `libs2/crypto/ccm/ccm.c`

```c
uint16_t CCM_encrypt_and_auth(
  const uint8_t *key,          // ← ENCRYPTION KEY (16 bytes)
  const uint8_t *nonce,        // ← NONCE (16 bytes, only 13 used)
  const uint8_t *aad,          // Additional Authenticated Data
  const uint32_t aad_len,
  uint8_t *text_to_encrypt,    // Input/output buffer
  const uint16_t text_to_encrypt_len
)
{
  uint8_t blocks[2][BLOCK_SIZE];  // Working buffers
  uint8_t auth_tag[AUTH_TAG_LEN]; // 8-byte MAC tag
  uint8_t b0[BLOCK_SIZE];
  uint8_t a_i[BLOCK_SIZE];
  uint8_t *ptr_to_msg = text_to_encrypt;
  
  // ===== STEP 1: Format B0 block (first block for CBC-MAC) =====
  format_b0(AUTH_TAG_LEN, text_to_encrypt_len, b0, nonce);
  // B0 = [Flags | Nonce | Length]
  
  memcpy(blocks[0], b0, BLOCK_SIZE);
  
  // ===== STEP 2: Compute CBC-MAC over AAD =====
  format_aad(blocks, aad, aad_len, key);
  // Authenticates: NodeIDs, HomeID, Length, S2 header
  
  // ===== STEP 3: Compute CBC-MAC over plaintext =====
  while (text_to_encrypt_len > 0) {
    bit_xor(ptr_to_msg, blocks[0], MIN(BLOCK_SIZE, remaining));
    ciph_block(blocks[0], key);  // ← AES-128 encryption
    ptr_to_msg += BLOCK_SIZE;
    remaining -= BLOCK_SIZE;
  }
  
  // blocks[0] now contains CBC-MAC result
  memcpy(auth_tag, blocks[0], AUTH_TAG_LEN);  // Take first 8 bytes
  
  // ===== STEP 4: Encrypt plaintext using CTR mode =====
  format_a_i(0, a_i, nonce);  // Counter block A_0
  
  int blocks_count = (text_to_encrypt_len + BLOCK_SIZE - 1) / BLOCK_SIZE;
  ptr_to_msg = text_to_encrypt;
  
  for (int i = 0; i < blocks_count; i++) {
    format_a_i(i + 1, a_i, nonce);  // A_i = [Flags | Nonce | Counter_i]
    ciph_block(a_i, key);           // ← Encrypt counter block
    
    // XOR with plaintext
    bit_xor(a_i, ptr_to_msg, MIN(BLOCK_SIZE, remaining));
    
    ptr_to_msg += BLOCK_SIZE;
  }
  
  // ===== STEP 5: Encrypt authentication tag =====
  format_a_i(0, a_i, nonce);  // A_0 again
  ciph_block(a_i, key);        // ← Encrypt A_0
  bit_xor(a_i, auth_tag, AUTH_TAG_LEN);  // Encrypted MAC
  
  // ===== STEP 6: Append encrypted MAC to ciphertext =====
  memcpy(text_to_encrypt + text_to_encrypt_len, auth_tag, AUTH_TAG_LEN);
  
  return text_to_encrypt_len + AUTH_TAG_LEN;  // Ciphertext + MAC
}
```

**AES-128 Block Cipher**:
```c
// libs2/crypto/aes/aes.c
void AES128_ECB_encrypt(
  const uint8_t* input,   // 16-byte block
  const uint8_t* key,     // ← ENCRYPTION KEY (16 bytes)
  uint8_t* output         // 16-byte encrypted block
)
{
  // Software AES-128 implementation
  // OR
  // Hardware crypto accelerator (if available)
}
```

---

## 4. Concrete Example Flow

### **Scenario**: Gateway gửi BASIC_SET(0xFF) đến Node 5 với S2_AUTHENTICATED

```
=== INITIAL STATE ===
Gateway: NodeID = 1
Node 5: NodeID = 5
HomeID: 0xAABBCCDD
Security: S2_AUTHENTICATED (class_id = 1)

Keys trong NVM:
- S2_AUTHENTICATED Key: 0x1234...ABCD (16 bytes)
- Nonce Key (derived):  0x5678...EF01 (16 bytes)

SPAN[Node 5]:
  state = SPAN_NEGOTIATED
  tx_seq = 42
  class_id = 1

---

=== STEP 1: Application Command ===
ClassicZIPNode_sendRequest() gọi từ controller:
  Payload: [0x20, 0x01, 0xFF]  // BASIC_SET, value=0xFF
  Destination: Node 5
  Security: S2_AUTHENTICATED

---

=== STEP 2: Transport Service ===
ZW_TransportService_SendData():
  p->scheme = SECURITY_SCHEME_2_AUTHENTICATED
  p->dnode = 5
  data = [0x20, 0x01, 0xFF]
  len = 3

→ Route to S2: sec2_send_data()

---

=== STEP 3: S2 Wrapper ===
sec2_send_data():
  s2_con.l_node = 1
  s2_con.r_node = 5
  s2_con.class_id = 1  // S2_AUTHENTICATED

→ Call libs2: S2_send_data(s2_ctx, &s2_con, data, 3)

---

=== STEP 4: S2 Check SPAN ===
S2_send_data_all_cast():
  Check SPAN[Node 5]: SPAN_NEGOTIATED ✓
  → Can encrypt immediately

→ Call: S2_encrypt_and_send()

---

=== STEP 5: ENCRYPTION (CHÍNH XÁC TẠI ĐÂY) ===

S2_encrypt_and_send():

A) Build S2 Header:
   msg[0] = 0x9F  // COMMAND_CLASS_SECURITY_2
   msg[1] = 0x03  // SECURITY_2_MESSAGE_ENCAPSULATION
   msg[2] = 42    // Sequence number
   msg[3] = 0x00  // Flags
   hdr_len = 4

B) Prepare payload:
   ciphertext = &msg[4]
   memcpy(ciphertext, [0x20, 0x01, 0xFF], 3)
   // ciphertext = [0x20, 0x01, 0xFF]

C) Build AAD:
   AAD = [
     0x01,           // Sender NodeID (gateway)
     0x05,           // Receiver NodeID (node 5)
     0xAA, 0xBB, 0xCC, 0xDD,  // HomeID
     0x00, 0x0F,     // Length (15 bytes total)
     0x2A, 0x00      // S2 header (seq=42, flags=0)
   ]
   // AAD length = 10 bytes

D) Generate Nonce:
   next_nonce_generate(&span->d.rng, nonce)
   // Uses NONCE KEY đã derive trước đó
   // Output: nonce = [0x7A, 0x8B, ..., 0x3C] (16 bytes)

E) READ ENCRYPTION KEY FROM MEMORY:
   uint8_t *enc_key = ctxt->sg[1].enc_key;
   // class_id = 1 (S2_AUTHENTICATED)
   
   ← KEY ĐÃ ĐƯỢC LOAD VÀO RAM TỪ NVM KHI INCLUSION
   
   enc_key = [0x12, 0x34, 0x56, ..., 0xAB, 0xCD]  (16 bytes)

F) AES-CCM ENCRYPTION:
   msg_len = CCM_encrypt_and_auth(
     enc_key,        // ← [0x12, 0x34, ..., 0xCD]
     nonce,          // ← [0x7A, 0x8B, ..., 0x3C]
     AAD,            // ← [0x01, 0x05, 0xAA, ...]
     10,             // AAD length
     ciphertext,     // ← [0x20, 0x01, 0xFF]
     3               // Plaintext length
   );
   
   // ===== INSIDE CCM_encrypt_and_auth =====
   
   1) Format B0:
      B0 = [0x49, nonce[0..12], 0x00, 0x03]
      
   2) CBC-MAC over AAD:
      X_0 = AES(key, B0)
      X_1 = AES(key, X_0 ⊕ AAD_block)
      
   3) CBC-MAC over plaintext:
      X_2 = AES(key, X_1 ⊕ [0x20, 0x01, 0xFF, 0x00...])
      MAC = X_2[0..7]  // 8 bytes
      
   4) CTR mode encryption:
      A_1 = [0x01, nonce[0..12], 0x00, 0x01]
      S_1 = AES(key, A_1)  // ← KEY ĐƯỢC SỬ DỤNG ĐỂ MÃ HÓA
      ciphertext[0..2] = plaintext[0..2] ⊕ S_1[0..2]
      
      Result: ciphertext = [0x7E, 0xA3, 0x2F]
      
   5) Encrypt MAC:
      A_0 = [0x01, nonce[0..12], 0x00, 0x00]
      S_0 = AES(key, A_0)
      encrypted_MAC = MAC ⊕ S_0[0..7]
      
      Result: encrypted_MAC = [0xB4, 0x6C, 0x91, 0xD8, 0x22, 0x5E, 0x7A, 0x3B]
      
   6) Append MAC:
      ciphertext = [0x7E, 0xA3, 0x2F, 0xB4, 0x6C, 0x91, 0xD8, 0x22, 0x5E, 0x7A, 0x3B]
      //            ^^^^^^^^^^^^^^^^^^  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
      //            Encrypted payload   Encrypted MAC (8 bytes)
      
   // msg_len = 11 (3 + 8)

G) Final S2 Packet:
   msg = [
     0x9F,                           // COMMAND_CLASS_SECURITY_2
     0x03,                           // SECURITY_2_MESSAGE_ENCAPSULATION
     0x2A,                           // Sequence = 42
     0x00,                           // Flags
     0x7E, 0xA3, 0x2F,              // Encrypted BASIC_SET
     0xB4, 0x6C, 0x91, 0xD8,        // MAC (part 1)
     0x22, 0x5E, 0x7A, 0x3B         // MAC (part 2)
   ]
   // Total length: 15 bytes

---

=== STEP 6: RF Transmission ===
S2_send_raw():
  → send_data_cb() (callback in S2_wrap.c)
  → ZW_SendDataEx(
      src_node = 1,
      dst_node = 5,
      data = [0x9F, 0x03, 0x2A, 0x00, 0x7E, ...],
      len = 15,
      tx_options = TRANSMIT_OPTION_ACK | TRANSMIT_OPTION_EXPLORE
    )
  → Serial API: 0x13 (FUNC_ID_ZW_SEND_DATA_EX)
  → Z-Wave chip RF modulator
  → Over the air: 915 MHz (US) or 868 MHz (EU)

---

=== RESULT ===
Node 5 receives encrypted packet:
  [0x9F, 0x03, 0x2A, 0x00, 0x7E, 0xA3, 0x2F, 0xB4, 0x6C, 0x91, 0xD8, 0x22, 0x5E, 0x7A, 0x3B]

Node 5 decrypts:
  1) Load S2_AUTHENTICATED key từ NVM
  2) Generate same nonce (synchronized RNG)
  3) AES-CCM decrypt with same key + nonce + AAD
  4) Verify MAC (8 bytes)
  5) Extract plaintext: [0x20, 0x01, 0xFF]
  6) Execute BASIC_SET(0xFF) → Turn on device

Gateway updates SPAN:
  SPAN[Node 5].tx_seq = 43  // Increment for next packet
```

---

## 5. Key Points - Tổng kết quan trọng

### **Thời điểm keys được sử dụng**:

| Key Type | Khi nào được đọc | Đọc từ đâu | Dùng để làm gì |
|----------|------------------|------------|----------------|
| **Network Key (S2_AUTHENTICATED)** | Lúc inclusion hoặc gateway boot | NVM flash (offset security2_key[1]) | Derive nonce_key và enc_key |
| **Nonce Key** | Lúc SPAN sync (SPAN_SOS_REMOTE_NONCE) | Derived từ network key | Seed nonce RNG (next_nonce_instantiate) |
| **Encryption Key** | **Mỗi lần encrypt packet** | RAM (ctxt->sg[class_id].enc_key) | AES-CCM encryption trong CCM_encrypt_and_auth() |
| **Nonce (per-packet)** | **Mỗi lần encrypt packet** | Generated từ nonce RNG | AES-CCM nonce parameter |

### **Hàm chính sử dụng keys**:

```c
// 1. Đọc network key từ NVM
keystore_network_key_read(KEY_CLASS_S2_AUTHENTICATED, buf)
  → NVM address: security2_key[1]
  → Component: Z-Wave chip flash memory
  
// 2. Derive encryption key
kderiv(shared_secret, constant_ek, enc_key)
  → Stored in: ctxt->sg[class_id].enc_key (RAM)
  
// 3. Instantiate nonce RNG  
next_nonce_instantiate(&span->d.rng, ei_sender, ei_receiver, nonce_key)
  → Component: CTR-DRBG state machine
  
// 4. Generate nonce
next_nonce_generate(&span->d.rng, nonce)
  → Component: CTR-DRBG (deterministic)
  
// 5. AES-CCM encryption
CCM_encrypt_and_auth(enc_key, nonce, aad, aad_len, plaintext, len)
  → Component: AES-128 engine (software or hardware)
  → Line: libs2/crypto/ccm/ccm.c:xxx
  → enc_key sử dụng TẠI ĐÂY trong ciph_block(blocks, key)
```

### **Physical components**:

1. **NVM Storage**: Z-Wave 700 series internal flash (NVM3)
2. **AES Engine**: 
   - Software: `libs2/crypto/aes/aes.c`
   - Hardware (optional): PSA Crypto API với Secure Vault
3. **Random Generator**: CTR-DRBG seeded from `/dev/urandom`

---

## 6. Security Notes

- **Keys KHÔNG BAO GIỜ được gửi qua RF** - chỉ có ciphertext
- **Nonce KHÔNG ĐƯỢC tái sử dụng** với cùng key (replay protection)
- **MAC authentication** ngăn chặn tampering
- **Sequence number** trong SPAN ngăn chặn replay attacks
- **ECDH key exchange** đảm bảo perfect forward secrecy trong inclusion

