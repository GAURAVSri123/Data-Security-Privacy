# WhatsApp End-to-End Encryption: Full Encryption and Decryption Architecture

Based on WhatsApp's public Security Whitepaper and the open Signal Protocol specs (X3DH, Double Ratchet). WhatsApp's app is closed source, so exact byte layouts may differ.

---

## 1. Components

| Layer | Purpose | Algorithms |
|---|---|---|
| Key generation | Create identity and pre-keys | Curve25519 (X25519) |
| Authentication | Prove pre-keys are genuine | XEdDSA signature |
| Session setup | Agree on a shared secret while receiver is offline | X3DH (4x Diffie-Hellman + HKDF) |
| Key evolution | New key for every message | Double Ratchet (DH ratchet + HMAC chain) |
| Message encryption | Scramble content | AES-256-CBC + PKCS#7 |
| Integrity | Detect tampering | HMAC-SHA256 (encrypt-then-MAC) |
| Groups | One ciphertext for many members | Sender Keys + signatures |
| Media | Large files | Random media key + AES-256-CBC + HMAC |
| Transport | Phone to server link | Noise protocol (Curve25519, AES-GCM, SHA-256) |

---

## 2. Keys and where they live

```
PHONE (private, never leaves)          SERVER (public only)
-----------------------------          --------------------
Identity Key private   (IK)            Identity Key public
Signed Pre-Key private (SPK)           Signed Pre-Key public + signature
One-Time Pre-Keys private (OPK x100)   One-Time Pre-Keys public (each handed out once)
Session state: RK, CKs, CKr, ratchet pairs, skipped keys
```

Per-session keys: Ephemeral Key (EK), Root Key (RK), Chain Keys (CK), Message Keys (MK, one per message).

---

## 3. Overall architecture

```
+-----------------------------------------------------------------------------+
| STAGE 0: REGISTRATION                                                       |
| Phone makes IK, SPK, sig=Sign(IK_priv, SPK_pub), ~100 OPK                   |
| Uploads PUBLIC parts to WhatsApp server (key directory)                     |
+-----------------------------------------------------------------------------+
                                     |
                                     v
+-----------------------------------------------------------------------------+
| STAGE 1: SESSION SETUP (X3DH), once per pair of devices                     |
| Alice fetches Bob's bundle -> verifies signature -> makes EK                |
| DH1..DH4 -> HKDF -> shared secret SK = initial Root Key                     |
+-----------------------------------------------------------------------------+
                                     |
                                     v
+-----------------------------------------------------------------------------+
| STAGE 2: PER-MESSAGE KEYS (Double Ratchet)                                  |
| DH ratchet (on speaker change) refreshes Root Key and Chain Keys            |
| Chain ratchet (every message) yields a unique Message Key                   |
+-----------------------------------------------------------------------------+
                                     |
                                     v
+-----------------------------------------------------------------------------+
| STAGE 3: ENCRYPT (sender)                                                   |
| plaintext -> pad -> AES-256-CBC -> HMAC-SHA256 -> packet                    |
+-----------------------------------------------------------------------------+
                                     |
                                     v
+-----------------------------------------------------------------------------+
| STAGE 4: TRANSPORT                                                          |
| Sender ==Noise/AES-GCM==> Server (stores/forwards ciphertext) ==> Receiver  |
+-----------------------------------------------------------------------------+
                                     |
                                     v
+-----------------------------------------------------------------------------+
| STAGE 5: DECRYPT (receiver)                                                 |
| parse header -> DH ratchet if new key -> derive MK -> verify MAC -> decrypt |
+-----------------------------------------------------------------------------+
```

Envelope view:

```
[ Noise transport layer (phone <-> server)
    [ Signal E2E ciphertext (Alice <-> Bob only)
        [ "Hello Bob" ]
    ]
]
```

The server can remove the transport layer but only ever sees the inner ciphertext.

---

## 4. Stage 0: Registration

```
BOB'S PHONE                                   SERVER
1. IK  = X25519 keypair (permanent)
2. SPK = X25519 keypair (rotated periodically)
3. sig = XEdDSA_Sign(IK_priv, SPK_pub)
4. OPK_1..OPK_100 = X25519 keypairs
5. Upload IK_pub, SPK_pub, sig, OPK_pubs  -->  stores bundle
   Private keys stay in device secure storage
```

When the OPK stock runs low, the phone uploads a fresh batch. The SPK is rotated periodically.

---

## 5. Stage 1: Session setup (X3DH)

### Sender (Alice)

```
1. Request Bob's bundle: {IK_B, SPK_B, sig, OPK_B}  (server deletes that OPK)
2. Verify XEdDSA(IK_B, SPK_B, sig). If invalid: abort.
3. Generate ephemeral pair EK_A.
4. DH1 = DH(IK_A_priv, SPK_B_pub)   -> authenticates Alice
   DH2 = DH(EK_A_priv, IK_B_pub)    -> authenticates Bob
   DH3 = DH(EK_A_priv, SPK_B_pub)   -> forward secrecy
   DH4 = DH(EK_A_priv, OPK_B_pub)   -> extra forward secrecy, replay protection
5. SK = HKDF( 0xFF*32 || DH1 || DH2 || DH3 || DH4 )
6. Send PreKeySignalMessage:
   { IK_A_pub, EK_A_pub, SPK_id, OPK_id, first Double-Ratchet message }
7. Delete EK_A_priv and the DH outputs.
```

### Receiver (Bob)

```
1. Read IK_A_pub, EK_A_pub, SPK_id, OPK_id from the message.
2. DH1 = DH(SPK_B_priv, IK_A_pub)
   DH2 = DH(IK_B_priv,  EK_A_pub)
   DH3 = DH(SPK_B_priv, EK_A_pub)
   DH4 = DH(OPK_B_priv, EK_A_pub)
3. SK = HKDF(0xFF*32 || DH1 || DH2 || DH3 || DH4)   (identical to Alice's)
4. Delete OPK_B_priv. Initialise Double Ratchet with SK.
```

Why it works: `DH(a_priv, B_pub) == DH(b_priv, A_pub)`. Both sides compute the same values without sending them.

---

## 6. Stage 2: Double Ratchet (key evolution)

### 6.1 State

```
RK    Root Key (32 bytes)
CKs   sending chain key        CKr  receiving chain key
DHs   my current ratchet pair  DHr  their latest ratchet public key
Ns/Nr message counters         PN   length of my previous sending chain
MKSKIPPED  saved keys for out-of-order messages (bounded)
```

### 6.2 DH ratchet (asymmetric, on speaker change)

```
(RK', CK) = HKDF( salt=RK, input=DH(my_priv, their_pub), info="WhisperRatchet" )
```

New random key pairs are mixed in, so a stolen old state cannot follow. This gives post-compromise security.

### 6.3 Chain ratchet (symmetric, every message)

```
MK_seed = HMAC-SHA256(CK, 0x01)
CK_next = HMAC-SHA256(CK, 0x02)     (old CK deleted)

CK0 --> CK1 --> CK2 --> CK3 ...
 |       |       |
MK0     MK1     MK2
```

HMAC is one-way, so old keys cannot be recomputed. This gives forward secrecy.

### 6.4 Message key expansion

```
HKDF(MK_seed, info="WhisperMessageKeys", 80 bytes)
  bytes  0-31  AES-256 key
  bytes 32-63  HMAC key
  bytes 64-79  IV (16 bytes)
```

### 6.5 Conversation timeline

```
ALICE                                               BOB
new pair a1; (RK1,CKs)=KDF_RK(RK, DH(a1,SPK_B))
A1 [a1_pub, n=0] ------------------------------->  new key a1_pub -> DH ratchet
A2 [a1_pub, n=1] ------------------------------->  chain step
A3 [a1_pub, n=2] ------------------------------->  chain step
                                                    new pair b1; (RK2,CKs)=KDF_RK(RK1, DH(b1,a1_pub))
<------------------------------ B1 [b1_pub, n=0]
DH ratchet with b1_pub
new pair a2 ...
A4 [a2_pub, n=0] ------------------------------->  DH ratchet
```

---

## 7. Stage 3: Encryption flow (sender)

```
INPUT: plaintext, CKs, Ns, my ratchet pub, PN

 1. seed      = HMAC-SHA256(CKs, 0x01)
 2. CKs       = HMAC-SHA256(CKs, 0x02)
 3. aes_key | mac_key | iv = HKDF(seed, "WhisperMessageKeys", 80)
 4. padded    = PKCS7_pad(plaintext)            (to 16-byte multiple)
 5. ciphertext= AES-256-CBC(aes_key, iv, padded)
 6. body      = version || Protobuf{ ratchet_pub, Ns, PN, ciphertext }
 7. mac       = HMAC-SHA256(mac_key, IK_sender || IK_receiver || body)[0:8]
 8. packet    = body || mac
 9. Ns        = Ns + 1
10. Zero out seed, aes_key, mac_key, iv
```

Packet layout:

```
+---------+--------------------------------------------------+----------+
| version | ratchet_pub | counter | prev_counter | ciphertext  | MAC (8B) |
+---------+--------------------------------------------------+----------+
   (header fields are not secret; they only tell the receiver which key to derive)
```

### AES-256-CBC internals

```
Block size 16 bytes, 14 rounds. Per round:
  SubBytes    substitute each byte via S-box
  ShiftRows   rotate rows
  MixColumns  mix bytes within each column
  AddRoundKey XOR with round key from key schedule
CBC chaining:
  C1 = AES(P1 XOR IV)
  C2 = AES(P2 XOR C1)
  C3 = AES(P3 XOR C2)
```

---

## 8. Stage 4: Transport

```
Phone                                  WhatsApp Server
  |-- ephemeral pub ------------------>|
  |<-- server ephemeral + static ------|   (Curve25519 DH mixed into a hash chain)
  |-- client static (encrypted) ------>|
  |=========== AES-GCM channel ========|
```

The server queues packets for offline users and delivers them when the recipient reconnects. It cannot read the inner Signal ciphertext.

---

## 9. Stage 5: Decryption flow (receiver)

```
INPUT: packet, session state

 1. Parse header: ratchet_pub, counter n, PN
 2. If ratchet_pub differs from the last one seen from the sender:
      a. Save skipped message keys for the old receiving chain (up to PN)
      b. DH ratchet: (RK, CKr) = HKDF(RK, DH(my_ratchet_priv, ratchet_pub))
      c. Generate my new ratchet pair for future replies
 3. If MKSKIPPED contains (ratchet_pub, n): use it, delete it, jump to step 5
 4. Advance CKr to counter n:
      for each skipped counter: seed = HMAC(CKr,0x01); CKr = HMAC(CKr,0x02);
                                store derived key in MKSKIPPED (bounded)
 5. seed = HMAC(CKr, 0x01); CKr = HMAC(CKr, 0x02)
    aes_key | mac_key | iv = HKDF(seed, "WhisperMessageKeys", 80)
 6. Verify MAC FIRST: HMAC(mac_key, IK_sender || IK_receiver || body)[0:8] == mac
      mismatch -> discard the message
 7. padded    = AES-256-CBC-Decrypt(aes_key, iv, ciphertext)
 8. plaintext = PKCS7_unpad(padded)
 9. Delete the message key. Display the message.
```

For a PreKeySignalMessage, Bob first runs the X3DH receiver steps (section 5) and then continues at step 1 above.

---

## 10. Key change summary

| Event | What changes |
|---|---|
| Every message | Chain key advances, new message key, old keys deleted |
| Speaker changes | New ratchet pair, new Root Key and chain keys |
| Session start | Fresh ephemeral key, OPK consumed and deleted |
| Periodic | Signed Pre-Key rotated |
| OPK stock low | New OPK batch uploaded |
| Reinstall or new phone | New Identity Key; contacts see "security code changed" |
| Group member leaves | Sender Keys regenerated and redistributed |

Forward secrecy: deleted message keys cannot be rebuilt from the current state.
Post-compromise security: the next new DH secret is unknown to a past attacker.

---

## 11. Out-of-order and lost messages

```
Received: msg5 before msg3, msg4
1. Step chain to counter 5, saving keys for 3 and 4 in MKSKIPPED
2. msg3 arrives: use saved key, delete it
3. msg4 arrives: use saved key, delete it
```

A cap on stored skipped keys prevents memory-exhaustion attacks. If a session cannot be recovered, the app shows "Waiting for this message" and a new session is negotiated.

---

## 12. Group chat architecture (Sender Keys)

### Setup (per sender, per group)

```
Alice generates: Chain Key (32B random) + signing keypair
Alice -> each member (via their 1:1 Signal session):
    SenderKeyDistributionMessage { key_id, chain_key, signing_pub }
```

### Send

```
seed = HMAC(ChainKey,0x01); ChainKey = HMAC(ChainKey,0x02)
aes_key, iv = HKDF(seed)
ct  = AES-256-CBC(aes_key, iv, plaintext)
sig = Sign(signing_priv, ct)
Upload ONE (ct, sig) to server -> server fans out to all members
```

### Receive

```
Look up sender's stored chain key -> derive same seed
Verify sig with signing_pub -> decrypt -> advance chain
```

```
Alice --1 ciphertext--> SERVER --copy--> Bob
                                --copy--> Carol
                                --copy--> Dave
```

Membership changes: on removal, all remaining members create new Sender Keys. New members receive current chain keys only, so they cannot read history.

---

## 13. Media architecture

```
SENDER
 1. media_key = random 32 bytes
 2. HKDF(media_key, type label) -> iv | aes_key | mac_key
 3. blob = AES-256-CBC(file) || HMAC
 4. Upload blob to media server -> URL
 5. Send E2E chat message { URL, media_key, SHA-256(blob), size, type }

RECEIVER
 1. Decrypt chat message (sections 9) -> get URL + media_key
 2. Download blob, verify hash and MAC
 3. Decrypt with keys from HKDF(media_key)
```

---

## 14. Multi-device architecture

```
Alice's phone --+--> encrypt for Bob's phone     (session 1)
                +--> encrypt for Bob's laptop    (session 2)
                +--> encrypt for Alice's laptop  (session 3)
```

Each device has its own Identity Key, pre-keys and sessions. Linking is done by QR scan, and the primary phone vouches for the new device with signatures. Senders encrypt once per recipient device.

---

## 15. Calls

Call setup messages travel over the E2E Signal sessions and carry a master secret. Keys derived from it encrypt audio and video with SRTP. Traffic goes peer to peer or via relays that see only encrypted packets.

---

## 16. Identity verification

The 60-digit security code and QR are derived from both users' Identity public keys. Matching codes prove no key substitution (man-in-the-middle) occurred. A changed Identity Key produces a changed code.

---

## 17. Encrypted backups

```
phone makes random 256-bit backup key
backup = encrypt(chat data, keys from backup key)
backup key protected by either:
   - 64-digit key you store yourself, or
   - password -> HSM Backup Key Vault (limited attempts)
```

---

## 18. What is not protected

Phone number, who talks to whom and when, IP address, message timing and size, some profile and group metadata. Also out of scope: malware on the device, screenshots or forwards by recipients, unencrypted cloud backups, and messages that a user reports.

---

## 19. Security properties

| Property | Provided by |
|---|---|
| Confidentiality | AES-256, keys never sent |
| Integrity | HMAC-SHA256, verified before decrypt |
| Authentication | Identity keys, signed pre-key, DH1/DH2 |
| Forward secrecy | Chain ratchet, deleted keys, OPKs |
| Post-compromise security | DH ratchet |
| Deniability (1:1) | MAC (shared key) rather than signature |

---

## 20. Algorithm cheat sheet

| Primitive | Role |
|---|---|
| X25519 | Diffie-Hellman shared secrets |
| XEdDSA | Sign and verify pre-keys |
| SHA-256 | Hashing, base for HMAC and HKDF |
| HMAC-SHA256 | MACs and one-way chain stepping |
| HKDF-SHA256 | Split one secret into many keys |
| AES-256-CBC | Message and media encryption |
| AES-GCM (Noise) | Transport channel |
| SRTP | Call media |

Further reading: WhatsApp Security Whitepaper (whatsapp.com/security), Signal specs at signal.org/docs (X3DH, Double Ratchet, XEdDSA), libsignal on GitHub.
