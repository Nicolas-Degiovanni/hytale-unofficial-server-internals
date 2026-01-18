---
description: Architectural reference for AuthToken
---

# AuthToken

**Package:** com.hypixel.hytale.protocol.packets.auth
**Type:** Transient Data Transfer Object (DTO)

## Definition
```java
// Signature
public class AuthToken implements Packet {
```

## Architecture & Concepts

The AuthToken class is a **Data Transfer Object** that represents a specific network message within the Hytale protocol layer, identified by PACKET_ID 12. It is not a service or a manager; its sole responsibility is to serve as a structured, in-memory representation of authentication credentials being exchanged between a client and server.

This class is a critical component of the initial connection and authentication sequence. It encapsulates the data required for a client to prove its identity to a server, such as an access token or a server-specific authorization grant.

Architecturally, AuthToken and its sibling packet classes embody a performance-oriented design choice to implement custom binary serialization. Instead of relying on reflection-based libraries, the class contains hand-optimized logic for writing to and reading from Netty's ByteBuf. This approach provides maximum control over the on-the-wire data format, minimizes overhead, and avoids garbage collection pressure in the performance-critical network pipeline.

The binary format itself is highly optimized, utilizing a leading bitmask (`nullBits`) to efficiently encode the presence or absence of nullable fields, followed by a block of fixed-size offsets that point to the location of variable-length data. This design minimizes payload size and allows for extremely fast, non-blocking parsing.

## Lifecycle & Ownership

-   **Creation:**
    -   **Outbound (Sending):** Instantiated directly by application logic, such as an authentication service, when preparing to send credentials to a remote endpoint. Example: `new AuthToken(accessToken, grant)`.
    -   **Inbound (Receiving):** Instantiated by the protocol's central packet deserialization pipeline. When the pipeline reads a packet header with ID 12, it invokes the static `AuthToken.deserialize` factory method to construct the object from the raw network buffer.
-   **Scope:** Extremely short-lived and transient. An AuthToken instance exists only for the brief moment it is being processed.
    -   An outbound packet is typically eligible for garbage collection immediately after it has been serialized into the network channel's buffer.
    -   An inbound packet is eligible for garbage collection as soon as the network event handler for that packet has completed its execution.
-   **Destruction:** Managed entirely by the Java garbage collector. There are no native resources or manual cleanup steps required.

## Internal State & Concurrency

-   **State:** AuthToken is a mutable data container. Its public fields can be modified after construction. However, by convention and design, instances should be treated as immutable after they are created or deserialized. Modifying a packet after it has been queued for sending or while it is being processed by a handler is an anti-pattern.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be created, serialized, deserialized, and processed within a single thread, typically a Netty I/O worker thread. Sharing an instance across multiple threads without explicit, external synchronization is unsafe and will lead to unpredictable behavior. The network framework's threading model guarantees that a single packet's lifecycle is confined to one thread, obviating the need for internal locking.

## API Surface

The public contract is dominated by static methods for validation and deserialization, and instance methods for serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static AuthToken | O(N) | Constructs an AuthToken by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the defined binary protocol. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a read-only check of a ByteBuf to ensure it contains a structurally valid AuthToken. Does not create an object. |
| computeSize() | int | O(1) | Calculates the exact number of bytes this object will occupy when serialized. Useful for buffer pre-allocation. |
| getId() | int | O(1) | Returns the constant packet identifier, 12. |

## Integration Patterns

### Standard Usage

The primary interaction pattern involves either creating an instance to be sent or receiving an instance within a network event handler.

```java
// Example: Sending an authentication request
AuthToken token = new AuthToken("client-access-token-xyz", null);

// The network layer will internally call token.serialize(buffer)
networkChannel.writeAndFlush(token);
```

```java
// Example: Handling a received token in a network handler
public void handleAuthToken(AuthToken token) {
    if (token.accessToken != null) {
        authenticationService.verify(token.accessToken);
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Instance Reuse:** Do not modify and resend the same AuthToken instance. Packets are designed to be single-use. Create a new instance for each message you intend to send.
-   **Cross-Thread Sharing:** Never pass an AuthToken instance received on a network I/O thread to a different worker thread without creating a deep copy. The underlying ByteBuf may be recycled, and concurrent access is not safe.
-   **Manual Serialization:** Do not attempt to read or write the binary format manually. The format is complex, involving bitmasks and relative offsets. Always rely on the provided `serialize` and `deserialize` methods to ensure protocol compliance.

## Data Pipeline

The AuthToken class is a data structure that moves through the network stack. It does not contain logic itself, but is acted upon by other components.

> **Outbound Flow (Client to Server):**
> Authentication Logic -> `new AuthToken(...)` -> Network Channel -> Packet Encoder -> **AuthToken.serialize()** -> TCP Socket

> **Inbound Flow (Server from Client):**
> TCP Socket -> ByteBuf -> Packet Decoder (sees ID 12) -> **AuthToken.deserialize()** -> AuthToken Instance -> Network Event Handler -> Authentication Logic

