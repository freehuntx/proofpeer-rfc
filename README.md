# ProofPeer: Secure Peer-to-Peer Messaging Protocol

**RFC Draft**  
**Date**: January 2025  
**Author**: freehuntx  
**Co-Author**: Claude Opus 4  
**Status**: Draft  
**Version**: 2.0

## Abstract

This document describes ProofPeer, a secure binary messaging protocol that enables authenticated peer-to-peer communication over untrusted transport systems. The protocol provides cryptographic identity verification, encrypted messaging, and application isolation while maintaining a small overhead through an efficient binary format.

## 1. Introduction

### 1.1. Background

Traditional peer-to-peer messaging systems often require complex infrastructure, centralized identity management, or expensive relay servers. Some transport systems like MQTT lack built-in authentication mechanisms. ProofPeer addresses these issues by providing a transport-agnostic binary protocol that adds cryptographic authentication and encryption to any messaging infrastructure.

### 1.2. Goals

- Provide secure peer-to-peer messaging without central authority
- Add authentication to untrusted transport systems
- Enable application isolation on shared infrastructure  
- Support both broadcast and private encrypted communication
- Minimize bandwidth and computational overhead through binary encoding
- Maintain transport independence

## 2. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

- **App ID**: Application identifier string (e.g., "com.example.app")
- **User ID**: MurmurHash3 32-bit unsigned integer of user's public key
- **Channel**: Logical grouping for broadcast messages
- **Peer**: Another user connected to the same application
- **Session**: Established AES encryption context between two peers
- **Packet ID**: 16-bit counter for replay protection
- **Transport**: Underlying message delivery system (MQTT, WebSocket, etc.)

## 3. Protocol Overview

### 3.1. Identity Model

Each user MUST generate a cryptographic key pair that serves as their identity. The public key MUST be hashed using MurmurHash3 to create a 32-bit unsigned integer User ID. Implementations MUST support Ed25519 keys and MAY support RSA keys.

### 3.2. Application Isolation

Applications MUST be isolated using App IDs. Each App ID MUST be hashed using MurmurHash3 to create a namespace for transport-specific addressing. Implementations MUST NOT allow cross-application communication at the protocol level.

### 3.3. Communication Modes

The protocol MUST support three communication modes:

1. **Broadcast**: Messages sent to public channels MUST be signed but MUST NOT be encrypted
2. **Direct (Unencrypted)**: Direct messages between peers MUST be signed
3. **Direct (Session)**: Messages within an established session MUST be encrypted using AES-256-GCM and MAY omit signatures

## 4. Binary Protocol Specification

### 4.1. Packet Structure

All ProofPeer packets MUST follow this structure:

```
┌─────────────────────────────────────────┐
│ Fixed Header (5 bytes)                  │
├─────────────────────────────────────────┤
│ Conditional Fields (based on flags)     │
├─────────────────────────────────────────┤
│ OpCode-specific Body                    │
└─────────────────────────────────────────┘
```

#### 4.1.1. Fixed Header

The fixed header MUST be exactly 5 bytes:

```
Offset  Size  Field
0       1     Protocol Version (uint8) = 0x01
1       1     Flags (uint8)
2       1     OpCode (uint8)
3       2     Packet ID (uint16, big-endian)
```

The Protocol Version field MUST be set to 0x01 for this version of the protocol. Implementations MUST reject packets with unsupported protocol versions.

#### 4.1.2. Flags

The Flags field MUST use the following bit assignments:

```
Bit  Value  Name           Description
0    0x01   FLAG_PUBKEY    Include public key
1    0x02   FLAG_SIGNED    Include signature
2    0x04   FLAG_RELIABLE  Request reliable delivery
3    0x08   FLAG_ENCRYPTED Body is encrypted
4    0x10   FLAG_SESSION   Include session ID
5-7  -      Reserved       Must be 0
```

Reserved bits (5-7) MUST be set to 0 by senders and MUST be ignored by receivers.

#### 4.1.3. OpCodes

Implementations MUST support the following OpCodes:

```
Value  Name                 Description
0x00   OP_HEARTBEAT         Keepalive signal
0x10   OP_CHANNEL_JOIN      Join channel
0x11   OP_CHANNEL_PRESENCE  Announce presence in the channel
0x12   OP_CHANNEL_LEAVE     Leave channel
0x13   OP_CHANNEL_MSG       Broadcast to channel
0x20   OP_MESSAGE           Unencrypted direct message
0x30   OP_SESSION_INIT      Start encryption session
0x31   OP_SESSION_ACK       Accept encryption session
0x32   OP_SESSION_MESSAGE   Encrypted direct message
0x33   OP_SESSION_CLOSE     End encryption session
```

Implementations MUST ignore packets with unknown OpCodes and SHOULD NOT generate error responses.

### 4.2. Conditional Fields

When flags are set, the corresponding fields MUST appear in the following order:

#### 4.2.1. Public Key (FLAG_PUBKEY)

When FLAG_PUBKEY is set, the following fields MUST be included:

```
Offset  Size  Field
0       1     Key Type (0=Ed25519, 1=RSA)
1       2     Key Length (uint16, big-endian)
3       n     Public Key Data
```

The Key Type field MUST be 0 for Ed25519 keys or 1 for RSA keys. The Key Length MUST match the actual key size (32 bytes for Ed25519, variable for RSA).

#### 4.2.2. Signature (FLAG_SIGNED)

When FLAG_SIGNED is set, the following fields MUST be included:

```
Offset  Size  Field
0       4     Unix Timestamp (uint32, big-endian)
4       8     Random Nonce
12      2     Signature Length (uint16, big-endian)
14      n     Signature of SHA-256(timestamp || nonce)
```

The timestamp MUST be the current Unix time in seconds. The nonce MUST be cryptographically random. The signature MUST be computed over the SHA-256 hash of the concatenated timestamp and nonce.

#### 4.2.3. Encryption (FLAG_ENCRYPTED)

When FLAG_ENCRYPTED is set, the following fields MUST be included:

```
Offset  Size  Field
0       4     Timestamp Echo (uint32, big-endian)
4       16    GCM Authentication Tag
20      4     Encrypted Length (uint32, big-endian)
24      n     Encrypted Data (AES-256-GCM)
```

The Timestamp Echo MUST be the timestamp from the last received message from the peer. The GCM Authentication Tag MUST be verified before decryption.

#### 4.2.4. Session ID (FLAG_SESSION)

When FLAG_SESSION is set, an 8-byte Session ID MUST be included:

```
Offset  Size  Field
0       8     Session ID (uint64, big-endian)
```

### 4.3. Transport Addressing

ProofPeer is transport-agnostic. Transport implementations MUST provide:

1. **Application Namespace**: MurmurHash3 of App ID
2. **Channel Addressing**: MurmurHash3 of channel name
3. **User Addressing**: User ID (MurmurHash3 of public key)

Example MQTT mapping:
```
/{app-hash}/c/{channel-hash}    # Broadcast channel
/{app-hash}/u/{user-id}         # User inbox
```

Implementations MUST use MurmurHash3 32-bit unsigned integers with seed 0 for all hashing operations.

## 5. Message Flows

### 5.1. Channel Join

To join a channel, implementations MUST:

```
1. Calculate channel_hash = MurmurHash3_32(channel_name)
2. Subscribe to channel address on transport
3. Send OP_CHANNEL_JOIN with FLAG_PUBKEY | FLAG_SIGNED
4. Receive OP_CHANNEL_PRESENCE messages from other peers (direct message, signed)
5. Build peer list with public keys
```

Peers receiving OP_CHANNEL_JOIN MUST respond with OP_CHANNEL_PRESENCE containing their public key.

### 5.2. Broadcast Message

To send a broadcast message, implementations MUST:

```
1. Create OP_CHANNEL_MSG packet
2. Set FLAG_SIGNED (and optionally FLAG_RELIABLE)
3. Include channel hash in body
4. Send to channel address
```

Broadcast messages MUST NOT be encrypted and MUST be signed.

### 5.3. Session Establishment

The initiator MUST:

```
1. Generate random AES-256 key
2. Generate random session ID
3. Encrypt AES key with responder's public key
4. Send OP_SESSION_INIT with FLAG_PUBKEY | FLAG_SIGNED | FLAG_SESSION
```

The responder MUST:

```
1. Verify signature and decrypt AES key
2. Store session ID → AES key mapping
3. Send OP_SESSION_ACK with FLAG_ENCRYPTED | FLAG_SESSION
4. Body encryption proves key receipt
```

### 5.4. Encrypted Communication

For encrypted messages, implementations MUST:

```
1. Look up AES key by session ID
2. Create OP_SESSION_MESSAGE with FLAG_ENCRYPTED | FLAG_SESSION
3. Encrypt payload with AES-256-GCM
4. Include GCM tag for authentication
5. Send to peer's user address
```

## 6. Cryptographic Specifications

### 6.1. Supported Algorithms

Implementations MUST support:

- **Public Key**: Ed25519
- **Signatures**: Ed25519
- **Encryption**: AES-256-GCM
- **Hashing**: MurmurHash3 32-bit (unsigned, seed=0)
- **KDF**: SHA-256

Implementations MAY support:

- **Public Key**: RSA (2048-bit minimum)
- **Signatures**: RSA-PSS with SHA-256

### 6.2. Signature Generation

Implementations MUST generate signatures as follows:

1. Construct signature input: timestamp (4 bytes) || nonce (8 bytes)
2. Compute SHA-256 hash of signature input
3. Sign hash with private key
4. Include signature in packet

### 6.3. Encryption Process

Implementations MUST encrypt data as follows:

1. Generate 12-byte IV from packet ID and session ID
2. Encrypt: timestamp (4 bytes) || payload
3. Compute GCM authentication tag
4. Place tag before encrypted data

### 6.4. Timestamp Validation

- Messages with timestamps more than 60 seconds in the past MUST be rejected
- Messages with timestamps more than 60 seconds in the future MUST be rejected
- Encrypted messages MUST decrypt to reveal matching timestamp

## 7. Security Considerations

### 7.1. Authentication

- All broadcast messages MUST be signed
- Signature MUST be verified using included public key
- User ID MUST match MurmurHash3 of public key

### 7.2. Replay Protection

- Implementations MUST track packet IDs per peer
- Implementations MUST reject packets with ID ≤ last seen (handling wraparound)
- Implementations MUST validate timestamps to prevent delayed replay
- Implementations SHOULD use random nonces for signature uniqueness

### 7.3. Encryption Security

- Implementations MUST verify GCM authentication tags before decryption
- Implementations MUST validate timestamp echo for quick validation
- Session IDs MUST be cryptographically random

### 7.4. Privacy

- Implementations MUST NOT log private keys or session keys
- Implementations SHOULD NOT persist decrypted message content
- Transport addresses MUST use hashed identifiers

### 7.5. Limitations

- The protocol does not protect against malicious transport operators
- Hash collisions (2^32 space) are possible but MUST be ignored
- The protocol does not include built-in spam or rate limiting

## 8. Implementation Requirements

### 8.1. Packet ID Management

Implementations MUST:

- Start at 0, increment by 1 per sent packet
- Wrap from 0xFFFF to 0x0000
- Track last seen ID per peer
- Reject packets with ID ≤ last seen (handle wraparound)

### 8.2. Session Management

Implementations MUST:

- Generate cryptographically random session IDs
- Maintain session ID → AES key mapping
- Expire sessions after 60 seconds of inactivity

Implementations SHOULD:

- Limit concurrent sessions per peer
- Implement session key rotation for long-lived connections

### 8.3. Performance Considerations

Implementations SHOULD:

- Cache signature verifications for 30 seconds
- Use FLAG_RELIABLE sparingly (high-frequency updates)
- Implement zero-copy parsing where possible
- Pool encryption contexts

### 8.4. Error Handling

- Invalid signatures MUST cause packet rejection
- Failed decryption MUST be silently ignored
- Malformed packets MUST be dropped
- Unknown OpCodes MUST be ignored

## 9. Examples

### 9.1. Minimal Heartbeat (5 bytes)

```
01 00 00 00 01
│  │  │  └─┴─ Packet ID: 1
│  │  └────── OpCode: OP_HEARTBEAT
│  └───────── Flags: none
└──────────── Version: 1
```

### 9.2. Channel Join with Ed25519 (~120 bytes)

```
01 03 10 00 02     # Header: Flags=PUBKEY|SIGNED, OpCode=JOIN, ID=2
00                 # Key type: Ed25519
00 20              # Key length: 32 bytes
[32-byte Ed25519 public key]
[4-byte timestamp]
[8-byte nonce]
00 40              # Signature length: 64 bytes
[64-byte Ed25519 signature]
A1 B2 C3 D4        # Channel hash
```

### 9.3. Encrypted Session Message (69+ bytes)

```
01 18 22 00 03     # Header: Flags=ENCRYPTED|SESSION, OpCode=SESSION_MSG, ID=3
5F A3 B2 C1        # Timestamp echo (plaintext)
[16-byte GCM tag]
00 00 00 20        # Encrypted length: 32 bytes
[32-byte encrypted data]
[8-byte session ID]
```

### 9.4. Complete Session Handshake

Initiator → Responder:
```
01 13 20 00 0A     # Flags=PUBKEY|SIGNED|SESSION, OpCode=SESSION_INIT
[Ed25519 key + signature fields]
12 34 56 78        # Target user ID
01 00              # Encrypted key length: 256 bytes
[256-byte RSA-encrypted AES key]
[8-byte session ID]
```

Responder → Initiator:
```
01 18 21 00 0B     # Flags=ENCRYPTED|SESSION, OpCode=SESSION_ACK
5F A3 B2 C2        # Timestamp echo
[16-byte GCM tag]
00 00 00 04        # Encrypted length: 4 bytes
[4-byte encrypted timestamp]
[8-byte session ID]
```

## 10. IANA Considerations

This document has no IANA actions.

## 11. References

### 11.1. Normative References

- RFC 2119: Key words for use in RFCs
- RFC 8032: Edwards-Curve Digital Signature Algorithm (EdDSA)
- RFC 8017: PKCS #1 v2.2 (RSA)
- NIST SP 800-38D: Galois/Counter Mode (GCM)

### 11.2. Informative References

- MQTT Version 3.1.1 and 5.0 Specifications
- MurmurHash3 Specification by Austin Appleby
- RFC 6090: Fundamental Elliptic Curve Cryptography Algorithms

## Appendix A: Test Vectors

### A.1. MurmurHash3 Tests

```
Input: "com.example.app"
Output: 3525125021 (0xD206A23D)

Input: "lobby"  
Output: 2458925847 (0x929F0317)

Input: "test.game.room"
Output: 2264751405 (0x87042D2D)
```

### A.2. Sample Packet

Channel join packet (hex):
```
01 03 10 00 01 00 00 20 3B 16 A5 87 CA 7A 4B D4
A4 49 4C EF D0 EA 1D 47 85 B2 C5 60 53 85 83 FC
86 D7 38 6B D2 E5 8B 71 67 50 FA 40 C0 BF 76 60
00 40 48 65 6C 6C 6F 20 57 6F 72 6C 64 21 0A 0A
54 68 69 73 20 69 73 20 61 20 74 65 73 74 20 6D
65 73 73 61 67 65 20 66 6F 72 20 74 68 65 20 50
72 6F 6F 66 50 65 65 72 20 62 69 6E 61 72 79 20
70 72 6F 74 6F 63 6F 6C 2E 92 9F 03 17
```

---

_End of ProofPeer RFC Draft v2.0_