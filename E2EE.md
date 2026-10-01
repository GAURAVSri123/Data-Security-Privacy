# 🔐 End-to-End Encryption (E2EE)

# Complete Technical Cryptographic Architecture

> A technically detailed architecture showing how two devices establish an encrypted session, derive keys, encrypt/decrypt messages, handle offline recipients, and continuously rotate keys.

---

# 1. Complete Cryptographic Architecture

```text
                           ┌──────────────────────────────┐
                           │         KEY SERVER           │
                           │                              │
                           │ Bob's Public Key Bundle      │
                           │                              │
                           │ IK_B_pub                     │
                           │ SPK_B_pub                    │
                           │ SPK_B_signature              │
                           │ OPK_B_pub                    │
                           │                              │
                           │ ❌ Bob's private keys        │
                           │ ❌ Session keys              │
                           │ ❌ Message plaintext         │
                           └──────────────┬───────────────┘
                                          │
                                          │ Public Key Bundle
                                          ▼
┌────────────────────────────────────────────────────────────────────┐
│                           ALICE DEVICE                             │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Identity Layer                                               │  │
│  │                                                              │  │
│  │ Ed25519 Identity Private Key  🔐                             │  │
│  │ Ed25519 Identity Public Key                                  │  │
│  └──────────────────────────────┬───────────────────────────────┘  │
│                                 │                                  │
│                                 ▼                                  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Session Establishment                                        │  │
│  │                                                              │  │
│  │ X25519                                                       │  │
│  │                                                              │  │
│  │ Identity Key + Ephemeral Key + Pre-Key Bundle                │  │
│  └──────────────────────────────┬───────────────────────────────┘  │
│                                 │                                  │
│                                 ▼                                  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Shared Secret                                                │  │
│  │                                                              │  │
│  │ DH1 || DH2 || DH3 || DH4                                     │  │
│  └──────────────────────────────┬───────────────────────────────┘  │
│                                 │                                  │
│                                 ▼                                  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ HKDF                                                         │  │
│  │                                                              │  │
│  │ Root Key / Initial Chain Keys                                │  │
│  └──────────────────────────────┬───────────────────────────────┘  │
│                                 │                                  │
│                                 ▼                                  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Double Ratchet                                               │  │
│  │                                                              │  │
│  │ Root Key → Chain Key → Message Key                           │  │
│  │                         ↓                                    │  │
│  │                    New Message Key                           │  │
│  └──────────────────────────────┬───────────────────────────────┘  │
│                                 │                                  │
│                                 ▼                                  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ AEAD                                                         │  │
│  │                                                              │  │
│  │ Plaintext + Key + Nonce + AAD                                │  │
│  │                    ↓                                         │  │
│  │          Ciphertext + Authentication Tag                     │  │
│  └──────────────────────────────┬───────────────────────────────┘  │
└─────────────────────────────────┼──────────────────────────────────┘
                                  │
                                  │ 🔒 CIPHERTEXT
                                  │
                                  ▼
                    ┌───────────────────────────┐
                    │          SERVER           │
                    │                           │
                    │ Message Router            │
                    │ Queue / Temporary Storage │
                    │                           │
                    │ 🔒 Ciphertext             │
                    │ 🔒 Header                 │
                    │ 🔒 Authentication Tag     │
                    │                           │
                    │ ❌ Cannot decrypt         │
                    └─────────────┬─────────────┘
                                  │
                                  │ 🔒 CIPHERTEXT
                                  ▼
┌────────────────────────────────────────────────────────────────────┐
│                            BOB DEVICE                              │
│                                                                    │
│  Receive encrypted packet                                          │
│             │                                                      │
│             ▼                                                      │
│  Identify session                                                  │
│             │                                                      │
│             ▼                                                      │
│  Double Ratchet state                                              │
│             │                                                      │
│             ▼                                                      │
│  Derive Message Key                                                │
│             │                                                      │
│             ▼                                                      │
│  AEAD Decryption + Authentication                                  │
│             │                                                      │
│             ▼                                                      │
│       Original Plaintext                                           │
└────────────────────────────────────────────────────────────────────┘
```

---

# 2. Cryptographic Building Blocks

The complete system contains multiple cryptographic primitives.

```text
                    E2EE CRYPTOGRAPHY
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
   Authentication      Key Agreement      Encryption
        │                  │                  │
        ▼                  ▼                  ▼
     Ed25519             X25519              AEAD
        │                  │                  │
        │                  ▼                  │
        │                 ECDH                │
        │                  │                  │
        │                  ▼                  │
        │                HKDF                 │
        │                  │                  │
        │                  ▼                  │
        │             Key Derivation          │
        │                                     │
        └─────────────────────────────────────┘
                           │
                           ▼
                    Double Ratchet
                           │
                           ▼
                 Forward Secrecy
              Post-Compromise Recovery
```

---

# 3. Algorithms Used

| Component           | Algorithm / Concept             | Purpose                            |
| ------------------- | ------------------------------- | ---------------------------------- |
| Identity signatures | Ed25519                         | Authenticate key material          |
| Key agreement       | X25519                          | Diffie-Hellman shared secrets      |
| KDF                 | HKDF-SHA-256                    | Derive independent keys            |
| Message encryption  | AEAD                            | Confidentiality + integrity        |
| Example AEAD        | AES-256-GCM / ChaCha20-Poly1305 | Encrypt messages                   |
| Ratchet             | Double Ratchet                  | Continuously change keys           |
| Hash                | SHA-256                         | Cryptographic hashing              |
| Randomness          | CSPRNG                          | Generate unpredictable keys/nonces |

---

# 4. Key Types

An E2EE device does **not** use a single key.

There are multiple key classes.

```text
DEVICE
 │
 ├── Identity Key Pair
 │      ├── Private
 │      └── Public
 │
 ├── Signed Pre-Key Pair
 │      ├── Private
 │      └── Public
 │
 ├── One-Time Pre-Key Pairs
 │      ├── OPK1
 │      ├── OPK2
 │      ├── OPK3
 │      └── ...
 │
 ├── Ephemeral Key Pair
 │      ├── Private
 │      └── Public
 │
 └── Session State
        │
        ├── Root Key
        ├── Sending Chain Key
        ├── Receiving Chain Key
        └── Message Keys
```

---

# 5. Identity Key

The identity key represents the device cryptographically.

For example:

```text
Alice:

IK_A_private
IK_A_public
```

Bob:

```text
IK_B_private
IK_B_public
```

The private key stays on the device.

```text
             ALICE DEVICE

      IK_A_private 🔐
             │
             │ NEVER SENT
             │
             X
             │
             ▼
          SERVER
```

The public key can be published:

```text
IK_A_public
```

---

# 6. Ed25519 — Identity Authentication

Ed25519 is used for digital signatures.

Suppose Bob has:

```text
IK_B_private
IK_B_public
```

Bob creates a signed pre-key:

```text
SPK_B_public
```

Bob signs it:

```text
signature_B =
Ed25519_Sign(
    IK_B_private,
    SPK_B_public
)
```

The server stores:

```text
IK_B_public
SPK_B_public
signature_B
```

Alice verifies:

```text
Ed25519_Verify(
    IK_B_public,
    signature_B,
    SPK_B_public
)
```

If verification succeeds:

```text
             Signature
                 │
                 ▼
        ┌─────────────────┐
        │ Is key authentic│
        │       ?         │
        └────────┬────────┘
                 │
          ┌──────┴──────┐
          │             │
         YES            NO
          │             │
          ▼             ▼
      Continue       Reject
```

---

# 7. Why Signatures Are Needed

Suppose the server sends Alice:

```text
Bob's Public Key
```

How does Alice know that the key really belongs to Bob?

Without authentication:

```text
Alice
  │
  │ Bob's public key?
  ▼
Server
  │
  │ Attacker's public key ❌
  ▼
Alice
```

A digital signature binds the signed pre-key to Bob's identity key.

---

# 8. Signed Pre-Key

Bob generates another X25519 key pair:

```text
SPK_B_private
SPK_B_public
```

Bob signs the public portion:

```text
Sig_B =
Sign(
    IK_B_private,
    SPK_B_public
)
```

Bob publishes:

```text
{
    IK_B_public,
    SPK_B_public,
    Sig_B
}
```

---

# 9. One-Time Pre-Key

Bob can also generate many one-time keys:

```text
OPK1
OPK2
OPK3
OPK4
...
OPKn
```

Each has:

```text
OPK_private
OPK_public
```

The server stores the public versions.

When Alice starts a session, one available OPK can be consumed.

```text
Server:

OPK1  → available
OPK2  → available
OPK3  → available
OPK4  → available

Alice requests bundle

OPK1 → assigned

Server:

OPK2
OPK3
OPK4
```

This provides additional asynchronous key material.

---

# 10. Why Pre-Keys Exist

Pre-keys solve an important problem:

> What if Bob is offline?

Alice can still establish session material using Bob's previously uploaded public keys.

```text
Alice                           Bob
  │                               │
  │                               │
  │                         📱 OFFLINE
  │                               │
  ▼
Server
  │
  │ Bob's public key bundle
  ▼
Alice
```

Alice does not need Bob to be online at that exact moment.

---

# 11. X25519

X25519 is an elliptic-curve Diffie-Hellman function.

Its purpose:

```text
Alice Private + Bob Public
             ↓
        Shared Secret
```

Bob independently calculates:

```text
Bob Private + Alice Public
             ↓
        Same Shared Secret
```

Conceptually:

```text
Alice:

X25519(
    Alice_private,
    Bob_public
)
        │
        ▼
     DH_SECRET


Bob:

X25519(
    Bob_private,
    Alice_public
)
        │
        ▼
     DH_SECRET
```

Both obtain the same secret.

---

# 12. X25519 Mathematical Idea

X25519 operates on Curve25519.

Conceptually:

```text
PublicKey = ScalarMult(
                PrivateKey,
                BasePoint
            )
```

Therefore:

```text
A = aG

B = bG
```

where:

```text
a = Alice private scalar
b = Bob private scalar
G = curve base point
```

Alice computes:

```text
aB
```

Since:

```text
B = bG
```

Alice gets:

```text
a(bG)
= abG
```

Bob computes:

```text
bA
```

Since:

```text
A = aG
```

Bob gets:

```text
b(aG)
= abG
```

Therefore:

```text
Alice Shared Secret = abG

Bob Shared Secret   = abG
```

---

# 13. X3DH-Style Session Establishment

A richer E2EE session setup uses multiple DH calculations.

Let:

```text
Alice Identity Key      = IK_A
Alice Ephemeral Key     = EK_A

Bob Identity Key        = IK_B
Bob Signed Pre-Key      = SPK_B
Bob One-Time Pre-Key    = OPK_B
```

The protocol can derive several DH values.

Conceptually:

```text
DH1 = X25519(IK_A_private, SPK_B_public)

DH2 = X25519(EK_A_private, IK_B_public)

DH3 = X25519(EK_A_private, SPK_B_public)

DH4 = X25519(EK_A_private, OPK_B_public)
```

The exact protocol specification determines the ordering, encoding, domain separation, and whether an OPK is present.

Then:

```text
SK = HKDF(
        DH1 || DH2 || DH3 || DH4
     )
```

This produces initial session material.

---

# 14. Why Multiple DH Operations?

Using multiple authenticated DH relationships provides stronger binding between:

```text
Alice's identity
        +
Alice's ephemeral key
        +
Bob's identity
        +
Bob's signed pre-key
        +
Bob's one-time pre-key
```

Conceptually:

```text
       Alice                       Bob

     IK_A  ──────────────────────  IK_B
       │                             │
       │                             │
     EK_A  ────────────────────── SPK_B
       │                             │
       │                             │
       └────────────────────────── OPK_B

                 │
                 ▼
             DH values
                 │
                 ▼
            HKDF / KDF
                 │
                 ▼
          Initial Session Key
```

---

# 15. Complete Initial Handshake

```text
ALICE                                         BOB
  │                                             │
  │                         Identity Key        │
  │                         Signed Pre-Key      │
  │                         One-Time Pre-Key    │
  │                             │               │
  │◄────────────────────────────┘               │
  │                                             │
  │ Generate Ephemeral Key                      │
  │                                             │
  │ EK_A                                        │
  │                                             │
  │ DH1 = X25519(IK_A, SPK_B)                   │
  │ DH2 = X25519(EK_A, IK_B)                    │
  │ DH3 = X25519(EK_A, SPK_B)                   │
  │ DH4 = X25519(EK_A, OPK_B)                   │
  │                                             │
  │            DH1 || DH2 || DH3 || DH4         │
  │                       │                     │
  │                       ▼                     │
  │                      HKDF                   │
  │                       │                     │
  │                       ▼                     │
  │                  Initial Key Material       │
  │                                             │
  │────────────────────────────────────────────►│
  │                                             │
  │                  Session Established        │
```

---

# 16. HKDF

HKDF means:

> HMAC-based Key Derivation Function

It is normally described as two stages:

```text
          Input Key Material
                  │
                  ▼
              HKDF-Extract
                  │
                  ▼
                  PRK
                  │
                  ▼
              HKDF-Expand
                  │
                  ▼
          Derived Key Material
```

---

# 17. HKDF-Extract

Conceptually:

```text
PRK = HMAC(
        salt,
        IKM
      )
```

where:

```text
IKM = Input Key Material
PRK = Pseudorandom Key
```

For the initial handshake:

```text
IKM =
DH1 || DH2 || DH3 || DH4
```

---

# 18. HKDF-Expand

Conceptually:

```text
OKM =
HKDF-Expand(
    PRK,
    info,
    desired_length
)
```

The `info` parameter can provide protocol/context separation.

For example:

```text
info = "E2EE session v1"
```

Then:

```text
PRK
 │
 ├──► Root Key
 │
 ├──► Initial Sending Chain Key
 │
 └──► Initial Receiving Chain Key
```

---

# 19. Why HKDF?

Do not directly use:

```text
DH_SECRET
```

for everything.

Instead:

```text
DH_SECRET
     │
     ▼
    HKDF
     │
     ├──── Root Key
     │
     ├──── Chain Key
     │
     ├──── Encryption Key
     │
     └──── Other protocol material
```

This creates independent cryptographic keys for different purposes.

---

# 20. Session State

After initialization, Alice and Bob maintain local session state.

Example:

```text
Session State
│
├── Root Key (RK)
│
├── Sending Chain Key (CKs)
│
├── Receiving Chain Key (CKr)
│
├── DH Ratchet Private Key
│
├── DH Ratchet Public Key
│
├── Previous Chain Length
│
└── Message Counters
```

This state is stored locally.

---

# 21. Double Ratchet

The Double Ratchet combines two types of ratchets:

```text
             DOUBLE RATCHET
                    │
          ┌─────────┴─────────┐
          │                   │
          ▼                   ▼
    Symmetric Ratchet     DH Ratchet
          │                   │
          ▼                   ▼
    Chain Key Updates    New DH Secret
          │                   │
          └─────────┬─────────┘
                    ▼
                Root Key
```

---

# 22. Symmetric-Key Ratchet

Suppose Alice has:

```text
Chain Key = CK0
```

She derives:

```text
Message Key 1 = KDF(CK0)
Chain Key 1  = KDF(CK0)
```

Then:

```text
CK0
 │
 ├────► Message Key 1
 │
 └────► CK1

CK1
 │
 ├────► Message Key 2
 │
 └────► CK2

CK2
 │
 ├────► Message Key 3
 │
 └────► CK3
```

Therefore:

```text
Message 1 → MK1
Message 2 → MK2
Message 3 → MK3
```

---

# 23. Chain Key Derivation

Conceptually:

```text
MK_i = HMAC(
          CK_i,
          "message"
       )

CK_(i+1) = HMAC(
              CK_i,
              "chain"
           )
```

The actual protocol uses specified KDF constructions and constants; these labels illustrate the separation of purposes.

---

# 24. Why Message Keys Are Different

Suppose:

```text
Message 1 → MK1
Message 2 → MK2
Message 3 → MK3
Message 4 → MK4
```

Even if an attacker somehow obtains:

```text
MK3
```

that should not automatically reveal:

```text
MK1
MK2
MK4
MK5
```

This is one of the important properties provided by a ratcheting design.

---

# 25. DH Ratchet

The second part is the DH ratchet.

Alice has:

```text
Alice DH private = a
Alice DH public  = A
```

Bob:

```text
Bob DH private = b
Bob DH public  = B
```

They calculate:

```text
DH = X25519(a, B)
```

and:

```text
DH = X25519(b, A)
```

The resulting DH output feeds into the root-key KDF.

---

# 26. Root Key Ratchet

Conceptually:

```text
Old Root Key
      +
New DH Output
      │
      ▼
     HKDF
      │
      ├──────────────► New Root Key
      │
      └──────────────► New Chain Key
```

Therefore:

```text
RK0
 │
 │ + DH1
 ▼
HKDF
 │
 ├──► RK1
 └──► CK1

RK1
 │
 │ + DH2
 ▼
HKDF
 │
 ├──► RK2
 └──► CK2
```

---

# 27. Double Ratchet Full Flow

```text
                    ROOT KEY
                       │
             ┌─────────┴─────────┐
             │                   │
             ▼                   ▼
       DH Ratchet           Chain Ratchet
             │                   │
             ▼                   ▼
        New Root Key          Chain Key
                                 │
                                 ▼
                         Message Key
                                 │
                                 ▼
                              AEAD
                                 │
                                 ▼
                           Ciphertext
```

---

# 28. AEAD Encryption

AEAD means:

> Authenticated Encryption with Associated Data

It provides:

```text
Confidentiality
       +
Integrity
       +
Authentication of ciphertext
```

Examples include:

```text
AES-256-GCM
ChaCha20-Poly1305
```

---

# 29. AEAD Inputs

An AEAD encryption operation conceptually looks like:

```text
Ciphertext, Tag =
AEAD_Encrypt(
    Key,
    Nonce,
    Plaintext,
    AssociatedData
)
```

Where:

```text
Key            = Message Key
Nonce           = Unique nonce
Plaintext       = Message
AssociatedData  = Authenticated but unencrypted metadata
```

---

# 30. AEAD Architecture

```text
                  MESSAGE
                     │
                     ▼
             ┌───────────────┐
             │   Plaintext   │
             └───────┬───────┘
                     │
                     │
Message Key ─────────┤
                     │
Nonce ───────────────┤
                     │
AAD ─────────────────┤
                     ▼
             ┌───────────────┐
             │      AEAD     │
             │   Encryption  │
             └───────┬───────┘
                     │
             ┌───────┴────────┐
             │                │
             ▼                ▼
        Ciphertext      Auth Tag
             │                │
             └───────┬────────┘
                     ▼
               Network Packet
```

---

# 31. Associated Data

Some information does not need to be encrypted but must be authenticated.

Example:

```text
AAD:

protocol_version
sender_device_id
receiver_device_id
message_number
ratchet_public_key
```

Then:

```text
Ciphertext = Encrypt(
    Key,
    Nonce,
    Plaintext,
    AAD
)
```

If an attacker modifies authenticated metadata:

```text
AAD_modified
```

authentication fails.

---

# 32. Nonce

A nonce is a value used by the AEAD construction.

For many AEAD schemes, nonce uniqueness is critical.

Conceptually:

```text
Message 1 → Nonce 1
Message 2 → Nonce 2
Message 3 → Nonce 3
```

Never casually reuse the same nonce with the same key where the algorithm forbids it.

A robust implementation should use the protocol's specified nonce-generation rules rather than inventing its own.

---

# 33. Authentication Tag

AEAD produces an authentication tag.

```text
Plaintext
    +
Key
    +
Nonce
    +
AAD
    │
    ▼
   AEAD
    │
    ├── Ciphertext
    │
    └── Authentication Tag
```

Bob verifies the tag during decryption.

If an attacker changes:

```text
Ciphertext
```

then:

```text
Authentication Verification
          │
          ▼
        FAILED
```

Bob rejects the message.

---

# 34. Complete Encryption Pipeline

```text
              "Hello Bob!"
                    │
                    ▼
              Plaintext
                    │
                    ▼
             Message Key
                    │
                    ▼
                 Nonce
                    │
                    ▼
                  AAD
                    │
                    ▼
          ┌─────────────────┐
          │      AEAD       │
          │    Encrypt      │
          └────────┬────────┘
                   │
             ┌─────┴─────┐
             │           │
             ▼           ▼
        Ciphertext     Tag
             │           │
             └─────┬─────┘
                   ▼
             Network Packet
```

---

# 35. Encrypted Packet

A message sent through the server can conceptually look like:

```json
{
    "version": 1,
    "sender_device": "A1",
    "recipient_device": "B1",
    "ratchet_public_key": "...",
    "message_number": 15,
    "previous_chain_length": 14,
    "nonce": "...",
    "ciphertext": "...",
    "authentication_tag": "..."
}
```

The exact wire format depends on the implementation.

---

# 36. What the Server Does

The server receives:

```text
Encrypted Packet
       │
       ▼
Validate transport/authentication
       │
       ▼
Identify recipient
       │
       ▼
Queue
       │
       ▼
Deliver
```

The server does **not** need:

```text
Message Key
Root Key
Chain Key
Private Identity Key
Plaintext
```

---

# 37. Server Architecture

```text
                         CLIENT
                           │
                           │ TLS transport
                           ▼
                 ┌─────────────────────┐
                 │   API / Gateway     │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │  Message Service    │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │   Message Queue     │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ Temporary Storage   │
                 └──────────┬──────────┘
                            │
                            ▼
                       Recipient
```

TLS may protect the connection between device and server, while E2EE protects the message content independently of the server.

---

# 38. Offline Bob

Suppose Bob is offline.

```text
ALICE                    SERVER                    BOB
  │                         │                       │
  │                         │                    OFFLINE
  │                         │                       │
  │ 🔒 Message              │                       │
  ├────────────────────────►│                       │
  │                         │                       │
  │                         │ Store ciphertext      │
  │                         │                       │
  │                         │                       │
  │                         │     Bob Online        │
  │                         │◄──────────────────────┤
  │                         │                       │
  │                         │ 🔒 Ciphertext         │
  │                         ├──────────────────────►│
  │                         │                       │
  │                         │                    Decrypt
```

The server simply holds the encrypted message.

---

# 39. Message Decryption

Bob receives:

```text
Ciphertext
Nonce
Authentication Tag
AAD
Ratchet Header
```

Bob performs:

```text
          Ciphertext
               │
               ▼
         Locate Session
               │
               ▼
       Process Ratchet State
               │
               ▼
       Derive Message Key
               │
               ▼
              AEAD
               │
         ┌─────┴─────┐
         │           │
      Invalid       Valid
         │           │
         ▼           ▼
       Reject     Plaintext
```

---

# 40. Complete Decryption Equation

Conceptually:

```text
Plaintext =
AEAD_Decrypt(
    MessageKey,
    Nonce,
    Ciphertext,
    AAD,
    AuthenticationTag
)
```

If authentication fails:

```text
Plaintext = ERROR
```

The application should not accept unauthenticated plaintext.

---

# 41. Forward Secrecy

Forward secrecy means that compromise of a current key should not automatically reveal old messages, assuming the protocol's deletion and ratchet assumptions are maintained.

Example:

```text
Message 1 → MK1
Message 2 → MK2
Message 3 → MK3
Message 4 → MK4
Message 5 → MK5
```

After processing:

```text
MK1 → deleted
MK2 → deleted
MK3 → deleted
MK4 → deleted
```

Only necessary current state remains.

Therefore compromise of later state does not simply give an attacker every previous message key.

---

# 42. Post-Compromise Recovery

The DH ratchet provides another important property.

Suppose:

```text
Attacker compromises current session state
```

Later, a fresh DH ratchet step occurs:

```text
New DH Private
      +
Peer's New DH Public
      │
      ▼
New DH Secret
      │
      ▼
HKDF
      │
      ▼
New Root Key
      │
      ▼
New Chain Keys
```

This can allow the session to recover confidentiality after the compromise, assuming the attacker no longer controls the endpoints and the protocol's assumptions hold.

---

# 43. Why Delete Old Keys?

Suppose Alice has:

```text
MK1
MK2
MK3
MK4
MK5
```

After using MK1:

```text
MK1 → securely discarded
```

Then:

```text
MK2 → used → discard
MK3 → used → discard
MK4 → used → discard
```

The implementation should minimize retention of obsolete sensitive material.

---

# 44. Replay Protection

Suppose an attacker captures:

```text
Message #15
```

and sends it again.

Bob's ratchet/session state tracks message numbers and skipped-key handling.

```text
Message #15
       │
       ▼
Already processed?
       │
   ┌───┴───┐
   │       │
  YES      NO
   │       │
   ▼       ▼
Reject   Process
```

A real protocol also has to handle out-of-order delivery.

---

# 45. Out-of-Order Messages

Suppose messages arrive:

```text
1
2
4
3
```

Message 3 arrived late.

The receiving implementation may temporarily retain **skipped message keys** for expected out-of-order messages.

Conceptually:

```text
Chain
 │
 ├── MK1 → processed
 ├── MK2 → processed
 ├── MK3 → stored temporarily
 └── MK4 → processed
```

When message 3 arrives:

```text
MK3
 │
 ▼
Decrypt
 │
 ▼
Delete MK3
```

A real implementation needs strict bounds and state-management rules for skipped keys.

---

# 46. Complete Key Lifecycle

```text
                KEY GENERATION
                     │
                     ▼
              Identity Keys
                     │
                     ▼
              Pre-Key Bundle
                     │
                     ▼
               Key Exchange
                     │
                     ▼
                X25519 DH
                     │
                     ▼
              Shared Secrets
                     │
                     ▼
                   HKDF
                     │
                     ▼
                 Root Key
                     │
                     ▼
               Chain Keys
                     │
                     ▼
               Message Keys
                     │
                     ▼
                 AEAD Key
                     │
                     ▼
                  Encrypt
                     │
                     ▼
                 Ciphertext
                     │
                     ▼
               Message Sent
                     │
                     ▼
               Message Key
                  deleted
```

---

# 47. Full Cryptographic Data Flow

```text
                    ┌──────────────────────┐
                    │      BOB DEVICE      │
                    └──────────┬───────────┘
                               │
                               │ Generate
                               ▼
                    ┌──────────────────────┐
                    │ Identity Key Pair    │
                    │ Ed25519              │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Signed Pre-Key       │
                    │ X25519               │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ One-Time Pre-Key     │
                    │ X25519               │
                    └──────────┬───────────┘
                               │
                               ▼
                           SERVER
                               │
                     Public Bundle
                               │
                               ▼
                    ┌──────────────────────┐
                    │     ALICE DEVICE     │
                    └──────────┬───────────┘
                               │
                               ▼
                    Generate Ephemeral Key
                               │
                               ▼
                         X25519 / DH
                               │
                               ▼
                  DH1 || DH2 || DH3 || DH4
                               │
                               ▼
                             HKDF
                               │
                               ▼
                         Root Key
                               │
                               ▼
                      Double Ratchet
                               │
                     ┌─────────┴─────────┐
                     │                   │
                     ▼                   ▼
                DH Ratchet         Symmetric Ratchet
                     │                   │
                     └─────────┬─────────┘
                               ▼
                         Chain Key
                               │
                               ▼
                        Message Key
                               │
                               ▼
                            AEAD
                               │
                               ▼
                        Ciphertext
                               │
                               ▼
                           SERVER
                               │
                         Store/Forward
                               │
                               ▼
                             BOB
                               │
                               ▼
                           Ratchet
                               │
                               ▼
                        Message Key
                               │
                               ▼
                            AEAD
                               │
                               ▼
                           Plaintext
```

---

# 48. Cryptographic Separation

A strong architecture separates the purposes of keys.

```text
                    KEY MATERIAL
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
 Identity             Session          Message
   Key                  Keys             Keys
    │                    │                 │
    ▼                    ▼                 ▼
Authentication       Ratcheting         AEAD
    │                    │                 │
    ▼                    ▼                 ▼
Signatures          Key Evolution      Encryption
```

Do not use one key for all purposes.

---

# 49. Public vs Private Information

## Can be public

```text
Identity Public Key
Signed Pre-Key Public Key
One-Time Pre-Key Public Key
Ephemeral Public Key
Ratchet Public Key
```

## Must remain secret

```text
Identity Private Key
Pre-Key Private Key
Ephemeral Private Key
Root Key
Chain Keys
Message Keys
Session Secrets
```

---

# 50. What the Attacker Sees

Suppose an attacker intercepts network traffic.

```text
Alice
  │
  │ 🔒 ciphertext
  ▼
ATTACKER
  │
  │ sees:
  │
  ├── Ciphertext
  ├── Nonce
  ├── Public protocol data
  └── Some metadata
```

But attacker does not automatically have:

```text
❌ Message Key
❌ Chain Key
❌ Root Key
❌ Identity Private Key
❌ Plaintext
```

Therefore:

```text
Captured Ciphertext
        +
No Secret Key
        │
        ▼
Cannot directly decrypt
```

---

# 51. What Happens if Ciphertext is Modified?

Original:

```text
Ciphertext A
Tag A
```

Attacker changes it:

```text
Ciphertext A'
Tag A
```

Bob executes:

```text
AEAD_Decrypt(
    key,
    nonce,
    ciphertext_A',
    AAD,
    tag_A
)
```

Authentication fails:

```text
Tag Verification
       │
       ▼
    FAILED
       │
       ▼
    REJECT
```

This protects message integrity.

---

# 52. What Happens if the Server is Malicious?

Even if the server attempts:

```text
Read message
Modify message
Replace ciphertext
Replay message
```

the cryptographic protocol is designed so that:

```text
Read plaintext
      ❌

Modify authenticated ciphertext
      ❌

Forge valid message
      ❌
```

The server can still potentially interfere with **delivery and availability**, because E2EE does not magically prevent a server from dropping, delaying, or blocking traffic.

---

# 53. E2EE vs TLS

These are different layers.

### TLS

```text
Alice ─────── 🔒 TLS ─────── Server
```

TLS protects the network connection.

### E2EE

```text
Alice ───── 🔒 E2EE ───── Server ───── 🔒 E2EE ───── Bob
```

The server remains outside the message decryption boundary.

Therefore:

```text
TLS:
Client ↔ Server

E2EE:
Client ↔ Client
```

Both can be used together.

---

# 54. Complete Security Layers

```text
┌───────────────────────────────────────────┐
│              APPLICATION                  │
│                                           │
│             "Hello Bob"                   │
├───────────────────────────────────────────┤
│              E2EE LAYER                   │
│                                           │
│     Double Ratchet + Message Key          │
├───────────────────────────────────────────┤
│              AEAD                         │
│                                           │
│      AES-GCM / ChaCha20-Poly1305          │
├───────────────────────────────────────────┤
│              TRANSPORT                    │
│                                           │
│                  TLS                      │
├───────────────────────────────────────────┤
│              NETWORK                      │
│                                           │
│                  TCP/IP                   │
└───────────────────────────────────────────┘
```

---

# 55. Complete Example

Alice sends:

```text
"Transfer ₹500"
```

### Step 1

Alice has:

```text
Plaintext =
"Transfer ₹500"
```

### Step 2

Current chain key:

```text
CK_10
```

### Step 3

Derive:

```text
MK_10 = KDF(CK_10)
CK_11 = KDF(CK_10)
```

### Step 4

Generate/use the protocol-defined nonce.

### Step 5

Encrypt:

```text
Ciphertext, Tag =
AEAD_Encrypt(
    MK_10,
    Nonce,
    "Transfer ₹500",
    AAD
)
```

### Step 6

Send:

```text
{
    ratchet_public_key,
    message_number,
    nonce,
    ciphertext,
    tag
}
```

### Step 7

Server stores/forwards:

```text
🔒 Ciphertext
```

### Step 8

Bob receives the packet.

### Step 9

Bob derives:

```text
MK_10
```

from the synchronized ratchet state.

### Step 10

Bob executes:

```text
AEAD_Decrypt(
    MK_10,
    Nonce,
    Ciphertext,
    AAD,
    Tag
)
```

### Step 11

Result:

```text
"Transfer ₹500"
```

### Step 12

Old message key is discarded according to the protocol's key-erasure policy.

---

# 56. Complete E2EE Architecture — Final Diagram

```text
                             ┌───────────────────────────────┐
                             │          KEY SERVER           │
                             │                               │
                             │  Bob Identity Public Key      │
                             │  Bob Signed Pre-Key           │
                             │  Bob Signature                │
                             │  Bob One-Time Pre-Key         │
                             │                               │
                             │  PUBLIC INFORMATION ONLY      │
                             └───────────────┬───────────────┘
                                             │
                                  Key Bundle │
                                             ▼
┌────────────────────────────────────────────────────────────────────────┐
│                              ALICE                                     │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ IDENTITY                                                         │  │
│  │ Ed25519                                                          │  │
│  │ IK_A_private 🔐                                                  │  │
│  │ IK_A_public                                                      │  │
│  └───────────────────────────────┬──────────────────────────────────┘  │
│                                  │                                     │
│                                  ▼                                     │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ AUTHENTICATION                                                   │  │
│  │ Verify Bob's signed pre-key                                      │  │
│  └───────────────────────────────┬──────────────────────────────────┘  │
│                                  │                                     │
│                                  ▼                                     │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ SESSION ESTABLISHMENT                                             │  │
│  │ X25519                                                            │  │
│  │                                                                   │  │
│  │ DH1 = X25519(IK_A_private, SPK_B_public)                          │  │
│  │ DH2 = X25519(EK_A_private, IK_B_public)                           │  │
│  │ DH3 = X25519(EK_A_private, SPK_B_public)                          │  │
│  │ DH4 = X25519(EK_A_private, OPK_B_public)                          │  │
│  └───────────────────────────────┬──────────────────────────────────┘  │
│                                  │                                     │
│                                  ▼                                     │
│                      DH1 || DH2 || DH3 || DH4                          │
│                                  │                                     │
│                                  ▼                                     │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ HKDF                                                             │  │
│  │                                                                  │  │
│  │ HKDF-Extract → PRK                                               │  │
│  │ HKDF-Expand  → Initial Session Material                          │  │
│  └───────────────────────────────┬──────────────────────────────────┘  │
│                                  │                                     │
│                                  ▼                                     │
│                           ROOT KEY (RK)                                │
│                                  │                                     │
│                                  ▼                                     │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ DOUBLE RATCHET                                                   │  │
│  │                                                                  │  │
│  │        DH Ratchet          Symmetric Ratchet                     │  │
│  │             │                      │                             │  │
│  │             └──────────┬───────────┘                             │  │
│  │                        ▼                                         │  │
│  │                  New Root Key                                    │  │
│  │                        │                                         │  │
│  │                        ▼                                         │  │
│  │                    Chain Key                                     │  │
│  │                        │                                         │  │
│  │                        ▼                                         │  │
│  │                   Message Key                                    │  │
│  └───────────────────────────────┬──────────────────────────────────┘  │
│                                  │                                     │
│                                  ▼                                     │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ AEAD ENCRYPTION                                                  │  │
│  │                                                                  │  │
│  │ Plaintext + Message Key + Nonce + AAD                            │  │
│  │                         │                                        │  │
│  │                         ▼                                        │  │
│  │              AES-256-GCM / ChaCha20-Poly1305                     │  │
│  │                         │                                        │  │
│  │              ┌──────────┴──────────┐                             │  │
│  │              ▼                     ▼                             │  │
│  │         Ciphertext              Tag                              │  │
│  └──────────────────────┬───────────────────────────────────────────┘  │
└─────────────────────────┼──────────────────────────────────────────────┘
                          │
                          │ 🔒 ENCRYPTED PACKET
                          ▼
                 ┌────────────────────────┐
                 │        SERVER          │
                 │                        │
                 │  Router                │
                 │  Queue                 │
                 │  Temporary Storage     │
                 │                        │
                 │  🔒 Ciphertext         │
                 │                        │
                 │  ❌ No Message Key     │
                 │  ❌ No Plaintext       │
                 │  ❌ No Root Key        │
                 └───────────┬────────────┘
                             │
                             │ 🔒 ENCRYPTED PACKET
                             ▼
┌────────────────────────────────────────────────────────────────────────┐
│                                BOB                                     │
│                                                                        │
│                        Receive Packet                                  │
│                              │                                         │
│                              ▼                                         │
│                       Session Lookup                                   │
│                              │                                         │
│                              ▼                                         │
│                      Double Ratchet                                    │
│                              │                                         │
│                              ▼                                         │
│                       Message Key                                      │
│                              │                                         │
│                              ▼                                         │
│                            AEAD                                        │
│                              │                                         │
│                     ┌────────┴────────┐                                │
│                     │                 │                                │
│                  Invalid            Valid                              │
│                     │                 │                                │
│                     ▼                 ▼                                │
│                  Reject           Plaintext                            │
│                                       │                                │
│                                       ▼                                │
│                              "Transfer ₹500"                           │
└────────────────────────────────────────────────────────────────────────┘
```

---

# 57. Cryptographic Flow in One Line

```text
Identity Authentication
        ↓
Ed25519
        ↓
X25519 Key Agreement
        ↓
Multiple DH Secrets
        ↓
HKDF
        ↓
Root Key
        ↓
Double Ratchet
        ↓
Chain Key
        ↓
Message Key
        ↓
AEAD
        ↓
Ciphertext + Authentication Tag
        ↓
Server
        ↓
Recipient
        ↓
Ratchet
        ↓
Message Key
        ↓
AEAD Verification + Decryption
        ↓
Plaintext
```

---

# 58. Security Properties

A correctly implemented protocol of this general design aims to provide:

```text
┌───────────────────────────────────────┐
│          SECURITY PROPERTIES          │
├───────────────────────────────────────┤
│                                       │
│ 🔐 Confidentiality                    │
│                                       │
│ 🛡️ Integrity                          │
│                                       │
│ 👤 Authentication of key material     │
│                                       │
│ 🔄 Forward Secrecy                    │
│                                       │
│ 🔄 Post-Compromise Recovery           │
│                                       │
│ 🚫 Replay Resistance                  │
│                                       │
│ 📦 Offline Message Support            │
│                                       | 
│ 🔑 Key Separation                     │
│                                       │
└───────────────────────────────────────┘
```

---

# 59. Most Important Concept

The complete architecture can be remembered as:

```text
              WHO ARE YOU?
                   │
                   ▼
               Ed25519
                   │
                   ▼
          ESTABLISH SECRET
                   │
                   ▼
               X25519
                   │
                   ▼
          DERIVE SAFE KEYS
                   │
                   ▼
                HKDF
                   │
                   ▼
          CONTINUOUSLY CHANGE
                   │
                   ▼
          DOUBLE RATCHET
                   │
                   ▼
            ENCRYPT MESSAGE
                   │
                   ▼
                 AEAD
                   │
                   ▼
              CIPHERTEXT
                   │
                   ▼
                SERVER
                   │
                   ▼
              RECIPIENT
                   │
                   ▼
            AEAD DECRYPT
                   │
                   ▼
              PLAINTEXT
```

# 🔥 Final Principle

```text
                    PRIVATE KEYS
                         │
                         │
                  NEVER LEAVE DEVICE
                         │
                         ▼
             ┌─────────────────────┐
             │   CRYPTOGRAPHIC     │
             │      PROTOCOL       │
             └──────────┬──────────┘
                        │
        ┌───────────────┼────────────────┐
        ▼               ▼                ▼
     Ed25519          X25519            HKDF
  Authentication   Key Agreement    Key Derivation
        │               │                │
        └───────────────┼────────────────┘
                        ▼
                 Double Ratchet
                        │
                        ▼
                  Message Keys
                        │
                        ▼
                       AEAD
                        │
                        ▼
                  🔒 CIPHERTEXT
                        │
                        ▼
                     SERVER
                        │
                        ▼
                  🔒 CIPHERTEXT
                        │
                        ▼
                       BOB
                        │
                        ▼
                    AEAD Verify
                        │
                        ▼
                    🔓 PLAINTEXT
```

> **The server transports encrypted data; the endpoints perform the cryptographic operations that establish and evolve the keys used to protect the message.**


<img width="1024" height="1536" alt="Image" src="https://github.com/user-attachments/assets/61d8966d-7dc9-48d0-8052-73ea10ba2371" />
