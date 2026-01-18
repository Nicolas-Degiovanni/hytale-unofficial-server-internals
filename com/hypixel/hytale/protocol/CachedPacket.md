---
description: Architectural reference for CachedPacket
---

# CachedPacket

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Wrapper

## Definition
```java
// Signature
public final class CachedPacket<T extends Packet> implements Packet, AutoCloseable {
```

## Architecture & Concepts
The CachedPacket is a high-performance wrapper class designed for network broadcast optimization. Its primary function is to pre-serialize a given Packet into a raw Netty ByteBuf. This pattern trades a small amount of memory for a significant reduction in CPU load when the same packet must be sent to numerous clients simultaneously.

Instead of repeatedly executing the serialization logic for each recipient, the system performs the serialization once, stores the result in the CachedPacket, and then performs a highly efficient raw byte copy for each subsequent send operation.

This class is a critical component in the protocol layer, acting as an immutable, point-in-time snapshot of a packet's serialized state. It fully integrates with the Netty pipeline by implementing the Packet interface, allowing it to be processed by network handlers as if it were a standard, non-cached packet. The implementation of AutoCloseable is a deliberate design choice to enforce strict resource management of the underlying ByteBuf.

## Lifecycle & Ownership
The lifecycle of a CachedPacket is short-lived and requires careful management to prevent resource leaks.

-   **Creation:** A CachedPacket is created exclusively via the static factory method **cache(Packet packet)**. The system component that invokes this method becomes the initial owner of the CachedPacket and is responsible for its eventual destruction.

-   **Scope:** The object is intended to exist only for the duration of a single broadcast operation. For example, it might be created at the beginning of a game tick, used to send updates to all relevant players, and then immediately destroyed before the next tick. It is not designed for long-term storage.

-   **Destruction:** The underlying ByteBuf is a pooled resource that **must** be released back to Netty's memory pool. This is accomplished by calling the **close()** method. Failure to call close will result in a severe memory leak. The AutoCloseable interface enables the use of try-with-resources blocks, which is the strongly recommended pattern for ensuring cleanup.

## Internal State & Concurrency
-   **State:** The internal state, including the packet type, ID, and the cached ByteBuf, is effectively immutable upon creation. The data within the ByteBuf should not be modified after caching.

-   **Thread Safety:** This class is not thread-safe and requires disciplined usage in a concurrent environment.
    -   The **serialize** method is safe to call from multiple threads (e.g., different Netty I/O threads) provided the underlying buffer has not been released.
    -   The **close** method is a mutating operation that decrements the buffer's reference count. It is not thread-safe and must only be called once by the owning thread.
    -   **WARNING:** A race condition between a thread calling **serialize** and another calling **close** will result in an unpredictable IllegalStateException. The owner must guarantee that all serialization operations are complete before closing the packet.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| cache(T packet) | static CachedPacket | O(N) | Factory method. Serializes the packet into an internal buffer. N is the size of the input packet. |
| serialize(ByteBuf buf) | void | O(M) | Copies the internal buffer's bytes to the target buffer. M is the size of the cached data. Throws IllegalStateException if the packet has been closed. |
| close() | void | O(1) | Releases the underlying ByteBuf resource. Critical for preventing memory leaks. |
| getId() | int | O(1) | Returns the ID of the original packet. |
| getPacketType() | Class | O(1) | Returns the Class of the original packet. |

## Integration Patterns

### Standard Usage
The correct usage pattern involves a try-with-resources block to guarantee that the **close** method is called, even in the event of an exception. This prevents memory leaks.

```java
// Create the original, stateful packet
PlayerPositionUpdatePacket update = new PlayerPositionUpdatePacket(player.getPosition());

// Use a try-with-resources block to cache and broadcast it
try (CachedPacket<PlayerPositionUpdatePacket> cachedUpdate = CachedPacket.cache(update)) {
    for (ClientConnection client : nearbyClients) {
        client.sendPacket(cachedUpdate);
    }
} // cachedUpdate.close() is automatically called here
```

### Anti-Patterns (Do NOT do this)
-   **Leaking Resources:** Failing to call **close()** is the most critical anti-pattern. This will exhaust Netty's buffer pool and crash the server.
    ```java
    // BAD: The buffer is never released
    CachedPacket update = CachedPacket.cache(somePacket);
    broadcaster.send(update);
    ```
-   **Use After Close:** Accessing the packet after it has been closed will result in a runtime exception.
    ```java
    // BAD: Throws IllegalStateException
    CachedPacket update = CachedPacket.cache(somePacket);
    update.close();
    channel.write(update);
    ```
-   **Nested Caching:** Attempting to cache a packet that is already a CachedPacket is explicitly forbidden and will throw an IllegalArgumentException.

## Data Pipeline
The CachedPacket acts as a serialization accelerator. It intercepts a standard Packet object, converts it to a raw buffer, and allows that buffer to be efficiently duplicated into multiple network channels.

> Flow:
> Game Logic Creates Packet Object → **CachedPacket.cache(packet)** → Internal ByteBuf Snapshot → Copied to multiple Network Channel Pipelines → TCP Socket

