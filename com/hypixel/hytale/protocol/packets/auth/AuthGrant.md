---
description: Architectural reference for AuthGrant
---

# AuthGrant

**Package:** com.hypixel.hytale.protocol.packets.auth
**Type:** Transient Data Object

## Definition
```java
// Signature
public class AuthGrant implements Packet {
```

## Architecture & Concepts
The AuthGrant class is a Data Transfer Object (DTO) representing a specific message within the Hytale network protocol. It is not a service or manager, but a pure data container used during the initial client-to-server authentication sequence. Its primary responsibility is to carry security tokens between endpoints.

This packet is a critical component of the connection handshake. It is designed for high-performance network I/O, bypassing common serialization frameworks like JSON or Protobuf in favor of a custom, manually-implemented binary format. The serialization and deserialization logic operates directly on Netty ByteBufs, employing techniques such as bitfields for null checks and relative offsets for variable-length string data to minimize payload size and processing overhead.

The existence of fields like authorizationGrant and serverIdentityToken strongly implies an integration with a centralized authentication authority, likely following a pattern similar to OAuth 2.0. The client receives a grant from an auth service and presents it to the game server, which can then validate it.

## Lifecycle & Ownership
- **Creation:** An AuthGrant instance is created under two distinct circumstances:
    1.  **On the sending endpoint:** It is instantiated directly using its constructor, for example: `new AuthGrant(grant, token)`. This object is then passed to the network layer for serialization.
    2.  **On the receiving endpoint:** It is instantiated by the protocol's packet dispatcher, which invokes the static `deserialize` factory method on a raw network buffer. Application code should never call `deserialize` directly.

- **Scope:** The object's lifetime is exceptionally short. It exists only within the scope of a single network event processing cycle. Once the relevant network handler has extracted the token data, the AuthGrant object is no longer referenced and becomes eligible for garbage collection.

- **Destruction:** Cleanup is managed automatically by the Java Garbage Collector. There are no native resources or manual cleanup steps required.

## Internal State & Concurrency
- **State:** The internal state is **Mutable**. The public fields `authorizationGrant` and `serverIdentityToken` can be modified after the object has been created. However, by convention, instances are treated as immutable value objects after initial construction or deserialization.

- **Thread Safety:** This class is **not thread-safe**. All fields are public and accessed without any synchronization primitives. This design assumes that a packet object will be confined to a single thread, typically a Netty I/O worker thread.

    **Warning:** Sharing an AuthGrant instance across multiple threads without external locking will lead to race conditions and undefined behavior. Do not pass this object to other threads; instead, extract its data into a thread-safe structure if multi-threaded processing is required.

## API Surface
The public contract is dominated by static methods for serialization and validation, reflecting its role as a DTO coupled to the network layer.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static AuthGrant | O(N) | Constructs an AuthGrant object by reading from a ByteBuf. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into a ByteBuf according to the defined binary protocol. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a read-only check of the binary data in a buffer to ensure structural validity without full deserialization. |
| computeSize() | int | O(N) | Calculates the exact number of bytes the packet will occupy when serialized. |
| getId() | int | O(1) | Returns the static network identifier for this packet type (11). |

*N represents the total length of the string data within the packet.*

## Integration Patterns

### Standard Usage
Developers typically interact with an already-deserialized AuthGrant object inside a network event handler. The primary use case is to extract the token data and pass it to an authentication service.

```java
// Example within a network event handler
public void handleAuthentication(AuthGrant packet) {
    if (packet.authorizationGrant == null) {
        // Handle error: client failed to provide a grant
        connection.disconnect("Missing authorization grant.");
        return;
    }

    // Pass the grant to a dedicated service for verification
    AuthenticationResult result = authService.verifyGrant(packet.authorizationGrant);
    if (!result.isSuccess()) {
        connection.disconnect("Invalid grant.");
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not modify and re-send an existing AuthGrant instance. Packets are considered ephemeral. For each new message, create a new instance to ensure state integrity.
- **Manual Deserialization:** Never call `AuthGrant.deserialize` from application-level logic. This method is strictly for use by the low-level packet dispatching system.
- **Ignoring Validation:** Bypassing the `validateStructure` check in the network pipeline can expose the server to denial-of-service attacks via malformed packets designed to cause deserialization errors or excessive memory allocation.

## Data Pipeline
The AuthGrant object is a transient data structure that represents a message as it moves from a raw byte stream to structured application data.

> Flow:
> Raw TCP Byte Stream -> Netty Channel Pipeline -> Packet Frame Decoder -> **AuthGrant.deserialize** -> AuthGrant Object -> Network Event Handler -> Authentication Service

