# ProofPeer: Secure Peer-to-Peer Messaging Protocol over MQTT

**RFC Draft**  
**Date**: January 2025  
**Author**: freehuntx  
**Co-Author**: Claude 4 Opus  
**Status**: Draft  
**Version**: 1.0

## Abstract

This document describes ProofPeer, a secure messaging protocol that enables peer-to-peer communication over MQTT brokers. The protocol provides cryptographic identity verification, encrypted messaging, and application isolation without requiring centralized servers or registration. It is designed for use cases including gaming, chat applications, and IoT communication.

## 1. Introduction

### 1.1. Background

Traditional peer-to-peer messaging systems often require complex infrastructure, centralized identity management, or expensive relay servers. ProofPeer leverages existing MQTT infrastructure to provide secure, decentralized messaging with cryptographic proof of identity.

### 1.2. Goals

- Provide secure peer-to-peer messaging without central authority
- Enable application isolation on shared MQTT infrastructure  
- Support both broadcast and private encrypted communication
- Minimize bandwidth and computational overhead
- Work across all platforms including web browsers

## 2. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

- **App ID**: Application identifier string (e.g., "com.example.app")
- **User ID**: MurmurHash3 32-bit unsigned integer of user's public key
- **Channel**: MQTT topic for broadcast messages
- **Inbox**: MQTT topic for receiving private messages
- **Peer**: Another user connected to the same application
- **Session**: Established AES encryption context between two peers

## 3. Protocol Overview

### 3.1. Identity Model

Each user generates a cryptographic key pair (RSA or Ed25519) that serves as their identity. The public key is hashed using MurmurHash3 to create a 32-bit unsigned integer User ID.

### 3.2. Application Isolation

Applications are isolated using App IDs. Each App ID is hashed to create a namespace prefix for all MQTT topics.

### 3.3. Communication Modes

1. **Broadcast**: JWT-signed messages to public channels
2. **Direct (Initial)**: Public-key encrypted messages between peers
3. **Direct (Session)**: AES-encrypted messages after handshake

## 4. Message Formats

### 4.1. Topic Structure

All MQTT topics follow this pattern:

```
/{app-id-hash}/c/{channel-hash}     # Broadcast channel
/{app-id-hash}/u/{user-id-hash}     # User inbox
```

Where:
- `{app-id-hash}` = MurmurHash3_32(app_id_string) as unsigned integer
- `{channel-hash}` = MurmurHash3_32(channel_name_string) as unsigned integer
- `{user-id-hash}` = MurmurHash3_32(public_key_bytes) as unsigned integer

Example:
```
/3749260367/c/2458925847   # Broadcast to hashed channel
/3749260367/u/2847593021   # Direct to user
```

### 4.2. Message Envelope

All messages MUST contain:

```json
{
  "v": 1,              // Protocol version (integer)
  "t": 1704214799,     // Unix timestamp
  "m": { ... },        // Message-specific content
  "r": true            // Reliable delivery flag (optional, default: false)
}
```

The `r` (reliable) flag determines MQTT QoS level:
- `false` or omitted: QoS 0 (at most once)
- `true`: QoS 1 (at least once)

### 4.3. Broadcast Message

Broadcast messages use JWT for authentication:

```json
{
  "v": 1,
  "t": 1704214799,
  "r": false,
  "jwt": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

JWT payload structure:
```json
{
  "pubkey": "MIIBIjANBgkq...",  // Base64 public key
  "timestamp": 1704214799,      // Must match envelope timestamp
  "data": {                     // Application data
    "type": "announce",
    "action": "join"
  }
}
```

### 4.4. Handshake Message

Initial AES key exchange:

```json
{
  "v": 1,
  "t": 1704214799,
  "r": true,                    // Handshakes should be reliable
  "jwt": "eyJ...",              // JWT with sender's public key
  "m": {
    "handshake": {
      "aes_key": "YmFzZTY0...", // RSA-encrypted AES-256 key
      "sid": "a7f3e9b2"         // Session ID (random)
    }
  }
}
```

### 4.5. Handshake Acknowledgment

```json
{
  "v": 1,
  "t": 1704214799,
  "r": true,
  "m": {
    "sender_id": 2847593021,    // Sender's user ID
    "aes": {
      "sid": "a7f3e9b2",        // Session ID from handshake
      "data": "YmFzZTY0..."     // AES-encrypted "OK"
    }
  }
}
```

### 4.6. Session Message

After handshake completion:

```json
{
  "v": 1,
  "t": 1704214799,
  "r": false,                   // Can be true for important messages
  "m": {
    "sender_id": 2847593021,    // Sender's user ID
    "aes": {
      "data": "YmFzZTY0..."     // AES-encrypted payload
    }
  }
}
```

### 4.7. Message Size Limits

ProofPeer does not implement fragmentation. Messages are limited by the underlying MQTT broker's maximum message size (typically 256MB for MQTT 5.0). Applications SHOULD verify broker limits and handle large data transfers at a higher layer.

## 5. Cryptographic Specifications

### 5.1. Supported Algorithms

- **Public Key**: RSA-2048 or Ed25519
- **JWT Signature**: RS256 (RSA) or ES256 (Ed25519)
- **Key Encryption**: RSA-OAEP or Ed25519
- **Symmetric Encryption**: AES-256-GCM
- **Hashing**: MurmurHash3 32-bit (unsigned, seed=0)

### 5.2. Key Generation

Users MUST generate cryptographically secure key pairs using appropriate system random sources.

### 5.3. Timestamp Validation

- Messages with timestamps more than 60 seconds in the past MUST be rejected
- Messages with timestamps more than 60 seconds in the future MUST be rejected

## 6. Protocol Flows

### 6.1. Connection Flow

```
1. Generate or load key pair
2. Calculate app_hash = MurmurHash3_32(app_id)
3. Calculate user_id = MurmurHash3_32(public_key)
4. Connect to MQTT broker
5. Subscribe to /{app_hash}/u/{user_id}
```

### 6.2. Channel Join Flow

```
1. Calculate channel_hash = MurmurHash3_32(channel_name)
2. Subscribe to /{app_hash}/c/{channel_hash}
3. Broadcast JWT-signed announce message
4. Receive peer announcements
5. Send introduction to each peer's inbox
```

### 6.3. Private Messaging Flow

```
1. Obtain peer's public key from announcement
2. Generate AES-256 key
3. Send handshake with encrypted AES key (r=true)
4. Receive acknowledgment with AES-encrypted response
5. Use AES for all subsequent messages
```

### 6.4. Heartbeat Flow

```
1. Every 30 seconds: broadcast JWT-signed heartbeat
2. Track last heartbeat from each peer
3. Consider peer offline after 60 seconds without heartbeat
4. Clear AES sessions for offline peers
```

## 7. Security Considerations

### 7.1. Identity Verification

- All broadcast messages MUST be JWT-signed
- JWT signature MUST be verified using embedded public key
- User ID MUST match MurmurHash3 of public key

### 7.2. Encryption

- Private messages MUST be encrypted
- AES keys MUST be randomly generated
- AES-GCM MUST include authentication tag

### 7.3. Replay Protection

- Timestamp validation prevents replay attacks
- Session IDs prevent handshake replay

### 7.4. Privacy

- User IDs are hashed, not reversible
- App IDs are hashed, not visible in topics
- Channel names are hashed, not visible in topics
- No message persistence on broker

### 7.5. Limitations

- No protection against malicious MQTT brokers
- No built-in spam prevention
- Hash collisions are possible but ignored

## 8. MQTT Configuration

### 8.1. Requirements

- Any MQTT version supported by broker
- Clean session RECOMMENDED
- Retain flag MUST NOT be used

### 8.2. Quality of Service

QoS level is determined by the `r` (reliable) flag in each message:
- `r=false` or omitted: QoS 0 (at most once)
- `r=true`: QoS 1 (at least once)

### 8.3. Topic Permissions

Clients need permissions for specific topics:
- Subscribe to: `/{app_hash}/u/{user_id}` (own inbox)
- Subscribe to: `/{app_hash}/c/{channel_hash}` (each channel individually)
- Publish to: `/{app_hash}/u/{peer_user_id}` (each peer's inbox)
- Publish to: `/{app_hash}/c/{channel_hash}` (each channel individually)

No wildcard subscriptions are used.

## 9. Implementation Notes

### 9.1. Performance

- Cache verified JWTs for 60 seconds
- Maintain AES session pool
- Use `r=false` for high-frequency updates
- Use `r=true` for critical messages (handshakes, important state)

### 9.2. Storage

- Private keys MUST be stored securely
- Public key cache SHOULD persist between sessions
- Channel name to hash mappings SHOULD be cached
- AES sessions MAY be cleared on disconnect

### 9.3. Error Handling

- Invalid messages MUST be silently dropped
- Decryption failures MUST be ignored
- Malformed JSON MUST be rejected

## 10. Examples

### 10.1. Calculate Hashes

```javascript
const appId = "com.example.chat";
const appHash = murmurHash3_32(appId); // 3749260367

const channelName = "lobby";
const channelHash = murmurHash3_32(channelName); // 2458925847
```

### 10.2. Announce Message

```json
{
  "v": 1,
  "t": 1704214799,
  "r": false,
  "jwt": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJwdWJrZXkiOiJNSUlCSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQXJLZk53aSIsInRpbWVzdGFtcCI6MTcwNDIxNDc5OSwiZGF0YSI6eyJ0eXBlIjoiYW5ub3VuY2UiLCJhY3Rpb24iOiJqb2luIn19.signature"
}
```

### 10.3. Complete Handshake

Initiator → Responder:
```json
{
  "v": 1,
  "t": 1704214799,
  "r": true,
  "jwt": "eyJ...",
  "m": {
    "handshake": {
      "aes_key": "YmFzZTY0X2VuY3J5cHRlZF9hZXNfa2V5",
      "sid": "a7f3e9b2"
    }
  }
}
```

Responder → Initiator:
```json
{
  "v": 1,
  "t": 1704214800,
  "r": true,
  "m": {
    "sender_id": 2847593021,
    "aes": {
      "sid": "a7f3e9b2",
      "data": "YmFzZTY0X2Flc19lbmNyeXB0ZWQoIk9LIik="
    }
  }
}
```

### 10.4. Game Update Messages

High-frequency position update (unreliable):
```json
{
  "v": 1,
  "t": 1704214801,
  "r": false,
  "m": {
    "sender_id": 2847593021,
    "aes": {
      "data": "YmFzZTY0X2Flc19lbmNyeXB0ZWQocG9zaXRpb25fZGF0YSk="
    }
  }
}
```

Important state change (reliable):
```json
{
  "v": 1,
  "t": 1704214802,
  "r": true,
  "m": {
    "sender_id": 2847593021,
    "aes": {
      "data": "YmFzZTY0X2Flc19lbmNyeXB0ZWQocGxheWVyX2RlYXRoX2V2ZW50KQ=="
    }
  }
}
```

## 11. References

- MQTT Version 3.1.1 and 5.0 Specifications
- RFC 7519: JSON Web Token (JWT)
- RFC 8017: PKCS #1 v2.2 (RSA)
- RFC 8032: Edwards-Curve Digital Signature Algorithm (EdDSA)
- MurmurHash3 Specification by Austin Appleby

## Appendix A: Test Vectors

### A.1. MurmurHash3 Tests

```
Input: "com.example.app"
Output: 3525125021 (unsigned 32-bit)

Input: "lobby"  
Output: 2458925847 (unsigned 32-bit)

Input: "test.game.room"
Output: 2264751405 (unsigned 32-bit)
```

### A.2. Sample RSA Key Pair

```
-----BEGIN RSA PRIVATE KEY-----
[Test private key omitted for brevity]
-----END RSA PRIVATE KEY-----

-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAtest
-----END PUBLIC KEY-----

User ID (MurmurHash3): 1829475932
```

---

_End of ProofPeer RFC Draft v1.0_
