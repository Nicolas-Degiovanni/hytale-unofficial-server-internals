---
description: Architectural reference for PasswordAccepted
---

# PasswordAccepted

**Package:** com.hypixel.hytale.protocol.packets.auth
**Type:** Transient

## Definition
```java
// Signature
public class PasswordAccepted implements Packet {
```

## Architecture & Concepts
The PasswordAccepted class is a protocol-level signal object, representing a specific event within the client-server authentication handshake. It functions as a "marker packet" sent from the server to the client to confirm that the credentials provided in a preceding request were successfully validated.

Its primary architectural role is to serve as an immutable, payload-free message. The entire meaning of the packet is conveyed by its type and its unique packet identifier (ID 16). Upon receipt, the client's network layer transitions the connection's state machine from an *authenticating* state to a fully *authenticated* state, unlocking further gameplay-related communication.

This class is a fundamental component of the protocol's low-level machinery, designed for efficiency and simplicity. It contains no data, requires zero bytes for its payload, and its serialization and deserialization processes are effectively no-ops.

### Lifecycle & Ownership
- **Creation:** An instance is created exclusively by the client's network protocol decoder, specifically via the static `deserialize` method. This occurs when an incoming network buffer is read and the packet ID is identified as 16. It is never instantiated by application-level logic.
- **Scope:** Extremely short-lived. The object exists only for the duration of its processing within a single network event loop tick. It is created, passed to a registered packet handler, and immediately becomes eligible for garbage collection.
- **Destruction:** Managed by the Java Garbage Collector. Due to its transient nature, it is typically reclaimed almost immediately after its corresponding handler completes execution.

## Internal State & Concurrency
- **State:** **Immutable**. This class is stateless. It contains no instance fields, and all instances of PasswordAccepted are functionally identical. Its behavior is defined entirely by its type.
- **Thread Safety:** **Inherently thread-safe**. As a stateless, immutable object, it can be safely passed between threads without any synchronization mechanisms. In practice, its lifecycle is confined to the single Netty I/O thread that decodes and handles the incoming network traffic.

## API Surface
The API surface is dictated by the Packet interface. The primary symbols are used by the protocol framework itself, not by high-level application code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier (16) for this packet type. |
| deserialize(buf, offset) | PasswordAccepted | O(1) | **WARNING:** Static factory method for framework use only. Creates an instance from a network buffer. |
| serialize(buf) | void | O(1) | **WARNING:** Framework method. Writes the packet to a buffer; this is a no-op as the packet has no payload. |

## Integration Patterns

### Standard Usage
The sole intended use of this class is to be received by a packet handler, which then triggers a state change in a session or connection manager. The handler acts upon the *receipt* of the packet, not on any data within it.

```java
// Hypothetical usage within a client-side packet handler
public void handlePacket(PasswordAccepted packet) {
    // The arrival of this packet type is the signal.
    // No data needs to be read from the 'packet' object itself.
    log.info("Authentication successful. Transitioning to next state.");
    clientSession.transitionToAuthenticated();
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Instantiation:** Never create an instance using `new PasswordAccepted()`. These packets should only ever originate from the network deserialization pipeline. Manually creating one serves no purpose and subverts the authentication flow.
- **Storing References:** Do not store instances of PasswordAccepted in long-lived objects. They are transient events, not state containers. Process them and let them be garbage collected.
- **Payload Inspection:** Do not attempt to read fields or data from this packet. The protocol definition guarantees it has a size of zero and carries no information beyond its type.

## Data Pipeline
PasswordAccepted represents a single, discrete step in the server-to-client authentication data flow.

> Flow:
> Server Authentication Logic -> Server Network Encoder -> TCP/IP Stream -> Client Network Decoder -> **PasswordAccepted Instance** -> Client Packet Handler -> Client Session State Update

