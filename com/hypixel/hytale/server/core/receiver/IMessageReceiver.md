---
description: Architectural reference for IMessageReceiver
---

# IMessageReceiver

**Package:** com.hypixel.hytale.server.core.receiver
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface IMessageReceiver {
```

## Architecture & Concepts
The IMessageReceiver interface defines a fundamental contract within the server's core messaging architecture. It establishes a standardized endpoint for message delivery, abstracting the underlying recipient from the message sender. Any class that implements this interface signals its capability to receive and process a Message object.

This design pattern decouples message-producing systems (such as network packet handlers or internal event dispatchers) from message-consuming entities (like player connections, server systems, or world entities). A sender does not need to know the concrete type of the receiver; it only needs a reference to an object that fulfills the IMessageReceiver contract. This promotes modularity, simplifies routing logic, and is critical for enabling testable components, as mock receivers can be easily substituted.

## Lifecycle & Ownership
As an interface, IMessageReceiver does not have its own lifecycle. Its lifecycle is entirely governed by the concrete classes that implement it.

- **Creation:** This interface is not instantiated. Concrete classes like PlayerConnection or SystemService implement it as part of their definition.
- **Scope:** The scope and lifetime of an IMessageReceiver are identical to the scope and lifetime of the implementing object. For example, an IMessageReceiver implemented by a player connection object will exist only for the duration of that player's session.
- **Destruction:** The contract is "destroyed" when the implementing object is garbage collected. No explicit cleanup is associated with the interface itself.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, any class that implements IMessageReceiver is expected to manage its own internal state.
- **Thread Safety:** **CRITICAL WARNING:** The IMessageReceiver contract provides no guarantees of thread safety. Implementations of this interface **must** be thread-safe. The sendMessage method can and will be invoked from various contexts, including high-performance network I/O threads (e.g., Netty event loops) and the main server game thread. Any implementation that modifies shared state must use appropriate concurrency controls such as locks, atomic variables, or concurrent collections to prevent race conditions and data corruption.

## API Surface
The public contract consists of a single method for dispatching messages.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| sendMessage(@Nonnull Message var1) | void | O(1) | Dispatches a non-null Message to the receiver for processing. The complexity of the underlying processing is implementation-dependent. |

## Integration Patterns

### Standard Usage
The typical pattern involves a routing or dispatching system obtaining a reference to a specific IMessageReceiver and invoking its sendMessage method.

```java
// Example: A router forwarding a message to a specific player
PlayerConnection player = sessionManager.getPlayerConnection(playerId);
if (player != null) {
    // PlayerConnection implements IMessageReceiver
    IMessageReceiver receiver = player;
    receiver.sendMessage(someMessage);
}
```

### Anti-Patterns (Do NOT do this)
- **Blocking Implementations:** Never implement sendMessage with long-running or blocking operations (e.g., database queries, file I/O). This method is often called from a critical, high-throughput thread. Blocking it will cause severe performance degradation and may stall the entire server network pipeline. Offload expensive work to a separate worker thread or job queue.
- **Assuming Single-Threaded Invocation:** Do not write an implementation that assumes sendMessage will only be called from the main server thread. Always design for concurrent access.
- **Passing Null Messages:** The @Nonnull annotation is a strict contract. Senders should never pass a null message, and robust receivers may throw a NullPointerException if this contract is violated.

## Data Pipeline
IMessageReceiver acts as a terminal node or a forwarding gateway in various data flows. It is the point where an abstract Message is delivered to a concrete logical owner.

> **Example Flow (Network to Player):**
> Inbound Network Packet -> Netty Channel Handler -> Packet Decoder -> **IMessageReceiver (implemented by PlayerConnection)** -> Player-specific Logic

