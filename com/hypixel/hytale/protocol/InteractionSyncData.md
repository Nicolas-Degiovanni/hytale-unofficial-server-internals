---
description: Architectural reference for InteractionSyncData
---

# InteractionSyncData

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class InteractionSyncData {
```

## Architecture & Concepts

The InteractionSyncData class is a fundamental Data Transfer Object within Hytale's network protocol layer. It serves as a structured container, or payload, for serializing and deserializing the complete state of a player-initiated interaction between the client and server. An "interaction" in this context is a complex game action that can span multiple ticks, such as charging a bow, mining a block, or executing a special ability.

This class is not a manager or a service; it is a pure data structure. Its primary architectural purpose is to provide a highly optimized, language-neutral binary representation of an in-game action.

The binary format is custom-designed for performance and consists of two main parts:

1.  **Fixed-Size Block:** A 157-byte block containing primitive types and fixed-size objects. This allows for extremely fast, direct-offset reads without parsing.
2.  **Variable-Size Block:** An area following the fixed block that contains variable-length data, such as maps and arrays. The fixed block contains integer offsets pointing to the start of each variable field within this block.

A key optimization is the **Nullable Bit Field**, a 2-byte bitmask at the start of the payload. Each bit corresponds to a nullable field within the structure. This avoids the overhead of sending a full byte or boolean for each null check, significantly compacting the packet size.

## Lifecycle & Ownership

-   **Creation:**
    -   **Sending Peer (e.g., Client):** An instance is created using its constructor (`new InteractionSyncData()`) and populated by a higher-level system, such as an InteractionManager, which captures the current state of the player's action.
    -   **Receiving Peer (e.g., Server):** An instance is exclusively created by the static `deserialize` method. This method is invoked by the network protocol handler when a packet containing interaction data is received and decoded.
-   **Scope:** The object is **transient and short-lived**. It is designed to exist only for the duration of a single network transaction. It is created, serialized, sent, deserialized, processed, and then becomes eligible for garbage collection.
-   **Destruction:** The object is managed by the Java Garbage Collector. There are no native resources or manual cleanup procedures required. Once all references to the instance are dropped, it is destroyed.

## Internal State & Concurrency

-   **State:** The state of an InteractionSyncData instance is **highly mutable**. All fields are public, allowing for direct, low-overhead modification. This design choice prioritizes performance within the single-threaded context of the game loop or network event loop.
-   **Thread Safety:** This class is **not thread-safe**. It is designed with the expectation that it will be created, populated, serialized, and deserialized within a single, controlled thread.

    **WARNING:** Sharing an InteractionSyncData instance across threads without explicit, external synchronization will lead to race conditions, data corruption, and unpredictable behavior. If data must be passed from a network thread to a game logic thread, either create a deep copy using the `clone` method or implement a proper thread-safe queuing mechanism.

## API Surface

The primary contract of this class is its binary layout and the static methods that operate on it. Direct field access is the intended way to manipulate its data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static InteractionSyncData | O(N) | Constructs a new instance by reading from the provided ByteBuf at a specific offset. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the defined binary protocol. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a non-deserializing check on a buffer to ensure offsets and lengths are valid. Crucial for security and preventing crashes from malformed packets. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Calculates the total number of bytes the object occupies in the buffer, including its variable-length fields. |
| clone() | InteractionSyncData | O(N) | Creates a deep copy of the object, including its internal collections. |

## Integration Patterns

### Standard Usage

The canonical use case involves a network handler decoding an incoming buffer into an InteractionSyncData object, which is then passed to the game logic for processing.

```java
// Executed within a Netty channel handler or similar network layer component
public void handleInteractionPacket(ByteBuf packetBuffer) {
    // Validate before deserializing to prevent errors from malformed data
    ValidationResult result = InteractionSyncData.validateStructure(packetBuffer, 0);
    if (!result.isOk()) {
        // Disconnect client or log error
        return;
    }

    InteractionSyncData data = InteractionSyncData.deserialize(packetBuffer, 0);

    // Pass the immutable data snapshot to the game thread for processing
    gameLogic.processPlayerInteraction(player, data);
}
```

### Anti-Patterns (Do NOT do this)

-   **State Persistence:** Do not hold references to InteractionSyncData objects for longer than a single tick or event-handling cycle. It represents a point-in-time snapshot, not a persistent state entity.
-   **Cross-Thread Modification:** Never modify an instance from one thread while another thread is reading it. For example, do not deserialize on a Netty thread and pass the same mutable instance to the main game thread without a defensive copy or other synchronization strategy.
-   **Ignoring Validation:** Bypassing `validateStructure` before deserializing data from an untrusted source (like a game client) exposes the server to potential buffer overflow exploits or denial-of-service attacks caused by malformed packets.

## Data Pipeline

The class serves as a pivotal point in the flow of interaction data between client and server.

> **Serialization Flow (Client to Server):**
> Player Action -> Game Logic -> **InteractionSyncData (Populated)** -> `serialize()` -> Netty ByteBuf -> Network Stack

> **Deserialization Flow (Server from Client):**
> Network Stack -> Netty ByteBuf -> Protocol Decoder -> `deserialize()` -> **InteractionSyncData (Instance)** -> Game Logic -> World State Update

