---
description: Architectural reference for ServerPlayerListUpdate
---

# ServerPlayerListUpdate

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Data Transfer Object (Packet)

## Definition
```java
// Signature
public class ServerPlayerListUpdate {
```

## Architecture & Concepts
The ServerPlayerListUpdate class is a low-level Data Transfer Object (DTO) that represents a specific, fixed-size network packet within the Hytale protocol. It is not a service or manager, but rather a structured data container that defines the wire format for a message sent from the server to the client.

Its primary architectural role is to act as a schema for updating the client-side representation of the player list for a given world. The class encapsulates the logic for serializing this data into a Netty ByteBuf for network transmission and deserializing it from a ByteBuf on the receiving end.

The design, characterized by public fields and static serialization helpers, prioritizes performance and low garbage collection overhead. The fixed size of 32 bytes (two 16-byte UUIDs) is indicative of a highly optimized protocol designed for efficiency. This object is a fundamental building block of the server-client state synchronization mechanism.

## Lifecycle & Ownership
- **Creation:** An instance is created under two primary circumstances:
    1.  **On the Server:** Instantiated by game logic (e.g., a World or SessionManager) when a player's connection status changes, triggering the need to notify clients.
    2.  **On the Client:** Instantiated by the network protocol decoder when a corresponding packet is received from the server. The static `deserialize` factory method is the designated entry point for this path.

- **Scope:** Transient and extremely short-lived. An instance of ServerPlayerListUpdate exists only for the duration of its processing within a single network event or game tick. It is designed to be a temporary carrier of data.

- **Destruction:** The object becomes eligible for garbage collection immediately after its data has been either written to the network socket (server-side) or used to update the client's game state (client-side). No long-term references should be held.

## Internal State & Concurrency
- **State:** Mutable. The class is a simple data holder with public, non-final fields (`uuid`, `worldUuid`). Its state is intended to be set once upon creation and then read by a consumer. It performs no caching and has no internal logic beyond its data representation.

- **Thread Safety:** **This class is not thread-safe.** It contains no locks or other concurrency primitives. It is designed to be created, populated, and processed within a single thread, such as a Netty I/O worker thread or the main game loop. Sharing an instance across threads without external synchronization will result in data corruption and undefined behavior.

## API Surface
The public contract is dominated by static methods for interacting with the network buffer, reinforcing its role as a protocol definition.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static ServerPlayerListUpdate | O(1) | Factory method. Reads 32 bytes from the buffer at the given offset and constructs a new instance. |
| serialize(ByteBuf) | void | O(1) | Writes the instance's two UUIDs (32 bytes total) into the provided buffer. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-check to ensure the buffer contains enough readable bytes to deserialize the packet. |
| computeSize() | int | O(1) | Returns the constant size of the packet, which is 32 bytes. |

## Integration Patterns

### Standard Usage
This object should be handled by the network layer. A packet processor receives a buffer, identifies the packet type, and uses the static `deserialize` method to create a usable object. This object is then typically passed to an event handler or a system responsible for managing the player list UI.

```java
// Executed within a Netty channel handler or packet processor
// Assumes 'buffer' is a ByteBuf containing the packet data

ValidationResult result = ServerPlayerListUpdate.validateStructure(buffer, offset);
if (result.isOk()) {
    ServerPlayerListUpdate update = ServerPlayerListUpdate.deserialize(buffer, offset);

    // Dispatch the update to the game state or UI thread
    gameContext.getEventBus().post(new PlayerListUpdatedEvent(update.uuid, update.worldUuid));
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Retention:** Do not store instances of ServerPlayerListUpdate in caches or as member variables in long-lived services. They represent a point-in-time event, not a persistent entity.

- **Manual Serialization:** Avoid manually reading from or writing to a ByteBuf. Always use the provided `serialize` and `deserialize` methods to ensure protocol correctness.

- **Cross-Thread Sharing:** Never pass an instance to another thread for modification. If data must be shared, extract the primitive UUIDs and pass those instead.

## Data Pipeline
The ServerPlayerListUpdate object is a transient artifact in the data flow between the server's game logic and the client's user interface.

> **Server-Side Flow:**
> Game Event (Player Joins/Leaves) -> Session Manager -> **new ServerPlayerListUpdate()** -> `serialize()` -> Network Encoder -> TCP Socket

> **Client-Side Flow:**
> TCP Socket -> Network Decoder -> `deserialize()` -> **ServerPlayerListUpdate instance** -> Event Bus -> Player List UI Handler -> UI Render

