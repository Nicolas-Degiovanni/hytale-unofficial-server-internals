---
description: Architectural reference for FailureReply
---

# FailureReply

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class FailureReply implements Packet {
```

## Architecture & Concepts
The FailureReply class is a specific message type, or *Packet*, within Hytale's low-level networking protocol. It serves a single, critical purpose: to communicate the failure of a requested operation from a server-side system back to the originating client. It is part of the **Asset Editor** protocol subsystem, as indicated by its package.

This class is not a service or manager; it is a pure data container. Its design is optimized for high-performance network serialization and deserialization. The structure is defined by a series of static constants (e.g., FIXED_BLOCK_SIZE, PACKET_ID) that dictate its precise binary layout on the wire.

The core architectural pattern is **Request-Reply with Correlation ID**. The `token` field acts as the correlation identifier, allowing an asynchronous client to match this failure response to a specific request it sent previously. This is essential in a non-blocking network environment where multiple requests can be in-flight simultaneously.

## Lifecycle & Ownership
- **Creation:** The lifecycle of a FailureReply object is fundamentally different on the server versus the client.
    - **Server-Side (Sending):** Instantiated directly via its constructor (`new FailureReply(...)`) within a service that handles an asset editor operation. This typically occurs inside a catch block or after a validation check fails. The newly created object is then passed to the network layer for serialization.
    - **Client-Side (Receiving):** Never instantiated with `new`. Instead, it is created by the network protocol layer, which calls the static `deserialize` factory method upon receiving a byte buffer with the corresponding packet ID (300).

- **Scope:** Extremely short-lived and transient. A FailureReply object exists only for the duration of a single network event processing cycle. It is created, its data is consumed by a handler, and it is then immediately eligible for garbage collection. It is never persisted or held in any long-term state.

- **Destruction:** Managed entirely by the Java Garbage Collector. There are no manual cleanup or `close` methods. Once all references to the object are dropped, typically at the end of an event handler method, it is reclaimed.

## Internal State & Concurrency
- **State:** The class is mutable, with public fields for `token` and `message`. However, it is intended to be treated as an immutable value object after its initial creation. Once constructed or deserialized, its state should not be altered.

- **Thread Safety:** **This class is not thread-safe.** It contains no locks or other concurrency primitives. It is designed to be created, serialized, deserialized, and processed by a single thread, typically a Netty I/O worker thread.

    **Warning:** Sharing a FailureReply instance across threads without explicit synchronization or creating a defensive copy via the `clone` method will result in undefined behavior and is a severe anti-pattern.

## API Surface
The public API is dominated by static methods for serialization and validation, reflecting its role as a protocol definition.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static FailureReply | O(N) | Constructs a FailureReply object by reading from a Netty ByteBuf at a given offset. N is the size of the message. |
| serialize(buf) | void | O(N) | Writes the object's state into a binary format in the provided Netty ByteBuf. |
| computeSize() | int | O(N) | Calculates the number of bytes this object will occupy when serialized. Used for buffer pre-allocation. |
| validateStructure(buffer, offset) | static ValidationResult | O(N) | Performs a read-only check to validate if a buffer contains a structurally correct FailureReply packet. Does not perform a full deserialization. |
| getId() | int | O(1) | Returns the static network identifier for this packet type (300). |

## Integration Patterns

### Standard Usage
A FailureReply is processed by a network event handler. The handler dispatches the packet to a system responsible for managing asynchronous operations, which uses the token to reject a pending Future or Promise.

```java
// Client-side network handler
public void handlePacket(Packet packet) {
    if (packet instanceof FailureReply reply) {
        // Retrieve the original request context using the token
        PendingRequest request = pendingRequestManager.get(reply.token);
        if (request != null) {
            // Reject the future/promise associated with the original request
            request.getFuture().completeExceptionally(new OperationFailedException(reply.message));
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Modification:** Do not modify the fields of a FailureReply object after it has been deserialized. It represents a point-in-time snapshot of a server-side event.
- **Client-Side Instantiation:** Never use `new FailureReply()` on the client. Client code should only ever receive these objects from the network layer.
- **Ignoring the Token:** The `token` is the most critical piece of data for the client. Processing the `message` without using the `token` to correlate it to a specific request can lead to incorrect error handling.

## Data Pipeline
The flow of data for a FailureReply is unidirectional from the server to the client, triggered by an error condition.

> Flow:
> Server-Side Operation Failure -> Service Logic Catches Exception -> **new FailureReply(token, message)** -> Network Encoder -> TCP/IP Stack -> ... (Network Transit) ... -> Client TCP/IP Stack -> Network Decoder -> **FailureReply.deserialize(buffer)** -> Packet Dispatcher -> Client-Side Request Manager -> UI Error Notification

