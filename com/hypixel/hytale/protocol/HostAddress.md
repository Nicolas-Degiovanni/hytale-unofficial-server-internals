---
description: Architectural reference for HostAddress
---

# HostAddress

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Transfer Object (DTO)

## Definition
```java
// Signature
public class HostAddress {
```

## Architecture & Concepts

The HostAddress class is a fundamental Data Transfer Object (DTO) within the Hytale network protocol layer. Its sole responsibility is to represent a network endpoint, encapsulating a hostname string and a port number. It is a pure data container, not a service or manager.

The design is heavily optimized for low-level network I/O operations. This is evident from its direct integration with Netty's ByteBuf and the inclusion of explicit serialization, deserialization, and size computation methods. The class itself defines its own binary wire format, specified by constants such as FIXED_BLOCK_SIZE and MAX_SIZE, making it a self-contained and portable component for network communication. It serves as the canonical representation for server addresses passed within protocol packets.

## Lifecycle & Ownership

-   **Creation:** HostAddress instances are created on-demand. They are typically instantiated by the protocol layer when deserializing an incoming packet via the static `deserialize` factory method. They may also be created by application logic that needs to specify a connection target, such as a server browser UI preparing to initiate a connection.
-   **Scope:** The lifetime of a HostAddress object is short and context-specific. It exists only for the duration of a single network operation or configuration task. It is not managed by any central registry and is intended to be passed by value.
-   **Destruction:** Instances are managed by the Java Garbage Collector. There are no native resources or explicit cleanup methods. Once all references to an instance are dropped, it becomes eligible for collection.

## Internal State & Concurrency

-   **State:** The internal state is **mutable**. The public fields `host` and `port` can be directly modified at any time after instantiation. This design prioritizes performance and ease of use over the safety of immutability.

-   **Thread Safety:** This class is **not thread-safe**. Direct exposure of its mutable fields makes it highly susceptible to race conditions if a single instance is shared and modified by multiple threads without external synchronization.

    **WARNING:** Treat instances of HostAddress as thread-local or effectively immutable after creation. If an instance must be passed between threads, it is safer to create a new copy using the copy constructor or the `clone` method.

## API Surface

The primary API surface is oriented around binary serialization and deserialization for network transport.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | HostAddress | O(N) | **Static Factory.** Deserializes a HostAddress from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Serializes the object's state into the provided ByteBuf according to the Hytale wire protocol. |
| computeSize() | int | O(N) | Calculates the exact number of bytes this object will consume when serialized. |
| validateStructure(buffer, offset) | ValidationResult | O(N) | **Static.** Performs a lightweight check on a buffer to verify if it contains a structurally valid HostAddress without full deserialization. |

*N refers to the length of the host string.*

## Integration Patterns

### Standard Usage

The class is intended to be used as a data container when interacting with the network protocol layer. The static factory `deserialize` is the primary entry point for handling incoming data.

```java
// Example: Deserializing a server address from an incoming network buffer
ByteBuf incomingPacket = ...;
HostAddress serverAddress = HostAddress.deserialize(incomingPacket, 0);

System.out.println("Received address for server: " + serverAddress.host);

// Example: Creating an address to be sent in an outgoing packet
ByteBuf outgoingPacket = ...;
HostAddress targetServer = new HostAddress("play.hytale.com", (short) 25565);
targetServer.serialize(outgoingPacket);
```

### Anti-Patterns (Do NOT do this)

-   **Shared Mutable State:** Never share a single HostAddress instance between multiple threads where at least one thread might modify it. This will lead to unpredictable behavior and data corruption.

    ```java
    // ANTI-PATTERN: Sharing a mutable instance across threads
    HostAddress sharedAddress = new HostAddress("server-a", (short)1234);
    new Thread(() -> sharedAddress.host = "server-b").start(); // Race condition
    new Thread(() -> connect(sharedAddress)).start(); // May connect to server-a or server-b
    ```

-   **Manual Deserialization:** Do not manually read fields from a buffer to populate a new HostAddress instance. The binary format includes variable-length integers (VarInt) for string prefixes, which are correctly handled only by the `deserialize` method.

## Data Pipeline

HostAddress acts as a data record that is either produced by or consumed by the serialization pipeline. It does not process data itself; it *is* the data.

**Ingress (Network to Game):**
> Flow:
> Netty ByteBuf -> **HostAddress.deserialize** -> HostAddress Instance -> Game Logic (e.g., Server List UI)

**Egress (Game to Network):**
> Flow:
> Game Logic -> HostAddress Instance -> **HostAddress.serialize** -> Netty ByteBuf

