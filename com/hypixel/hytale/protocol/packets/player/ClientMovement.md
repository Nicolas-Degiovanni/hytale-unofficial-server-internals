---
description: Architectural reference for ClientMovement
---

# ClientMovement

**Package:** com.hypixel.hytale.protocol.packets.player
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class ClientMovement implements Packet {
```

## Architecture & Concepts
The ClientMovement class is a fundamental Data Transfer Object (DTO) within Hytale's networking protocol. It is not a service or manager; it is a pure data structure designed to encapsulate a complete snapshot of a player's physical state for a single network tick.

Its primary architectural role is to serve as the standardized data contract for communicating player motion from the client to the server. The server receives a stream of these packets and uses them to update its authoritative simulation of the player's entity in the game world.

The design is heavily optimized for network performance and low-latency processing. Key characteristics include:

*   **Fixed-Size Layout:** The packet has a constant size of 153 bytes, simplifying buffer allocation and parsing on the server.
*   **Bitmask for Nulls:** A 2-byte bit field at the start of the payload efficiently encodes the presence or absence of nullable complex types like Position or Vector3d. This avoids the overhead of variable-size encoding for optional data.
*   **Direct Serialization:** The class implements its own `serialize` and `deserialize` methods that operate directly on Netty's ByteBuf. This high-performance approach bypasses slower, reflection-based serialization frameworks.

This packet is the primary mechanism by which the client-side physics simulation informs the server of its results.

## Lifecycle & Ownership
As a transient DTO, ClientMovement instances have an extremely short and well-defined lifecycle. They are message-scoped and should not be retained across ticks.

*   **Creation:**
    *   **Client-Side:** A new ClientMovement object is instantiated by the client's main game loop or input processing system on every tick where movement data needs to be sent. It is populated with the most current state from the local player entity.
    *   **Server-Side:** An instance is created exclusively by the network protocol layer when it receives a raw byte buffer with Packet ID 108. The static `deserialize` factory method is invoked to construct the object from the network data.

*   **Scope:** The object's scope is confined to the immediate task of data transmission or processing. On the client, it exists only until it is passed to the network writer and serialized. On the server, it exists only until its data has been consumed by the relevant game logic (e.g., the PlayerEntity update method).

*   **Destruction:** Instances are eligible for garbage collection immediately after use. There are no persistent references to ClientMovement objects within the engine.

## Internal State & Concurrency
*   **State:** The object's state is entirely mutable and consists of public fields representing player physics data. It is a simple data container with no internal logic, caching, or state management.

*   **Thread Safety:** **This class is not thread-safe.** It is designed for synchronous, single-threaded access. All fields are public and non-volatile. Accessing a ClientMovement instance from multiple threads without external locking will result in race conditions, memory visibility issues, and corrupted network data.

    **WARNING:** The object must be fully populated on the client's main thread before being handed off to the network thread for serialization. On the server, it must be processed entirely within the scope of a single network event loop thread or dispatched to a specific game world thread.

## API Surface
The public contract is dominated by static methods for network protocol integration and the instance methods for serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static ClientMovement | O(1) | **Primary Entry Point (Server).** Constructs a ClientMovement object from a raw network buffer at a given offset. |
| serialize(ByteBuf) | void | O(1) | **Primary Exit Point (Client).** Writes the object's state into a network buffer according to the fixed-size protocol. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a low-cost pre-check to ensure a buffer is large enough to contain the packet before attempting deserialization. |
| clone() | ClientMovement | O(1) | Creates a deep copy of the packet. Useful for debugging or state inspection without affecting the primary instance. |

## Integration Patterns

### Standard Usage
The intended use is for the network layer to create the object on the server and for the game loop to create it on the client.

**Client-Side Population and Transmission**
```java
// Executed in the client's main update loop
ClientMovement packet = new ClientMovement();
packet.absolutePosition = localPlayer.getPosition();
packet.velocity = localPlayer.getVelocity();
packet.lookOrientation = localPlayer.getLookDirection();
// ... populate all other relevant fields ...

networkManager.sendPacket(packet);
```

**Server-Side Deserialization and Processing**
```java
// Executed in a server-side network handler
// The buffer and offset are provided by the network protocol layer
ClientMovement packet = ClientMovement.deserialize(buffer, offset);

PlayerEntity targetPlayer = world.getPlayerForConnection(connection);
targetPlayer.getMovementController().applyClientUpdate(packet);
```

### Anti-Patterns (Do NOT do this)
-   **Object Re-use:** Do not cache and re-use a ClientMovement instance across multiple ticks to "optimize" performance. The objects are lightweight, and re-using them is a common source of bugs where stale data from a previous tick is accidentally transmitted. Always create a new instance.
-   **Asynchronous Modification:** Never modify a ClientMovement object on one thread while it is being serialized by the network layer on another. This will lead to a partially-written, corrupt packet being sent.
-   **Server-Side Instantiation:** The server must never use `new ClientMovement()`. Server-side instances must only originate from the `deserialize` method, which constructs them from client-provided data.

## Data Pipeline
The ClientMovement packet is a critical link in the state synchronization pipeline between client and server.

> **Flow:**
> Client Input → Client Physics Simulation → **ClientMovement** (Instantiation) → `serialize()` → Netty ByteBuf → Network → Server Netty ByteBuf → `deserialize()` → **ClientMovement** (Re-constitution) → Server Game Logic → Player Entity State Update

