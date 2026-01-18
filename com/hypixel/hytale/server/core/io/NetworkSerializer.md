---
description: Architectural reference for NetworkSerializer
---

# NetworkSerializer

**Package:** com.hypixel.hytale.server.core.io
**Type:** Utility

## Definition
```java
// Signature
@FunctionalInterface
public interface NetworkSerializer<Type, Packet> {
```

## Architecture & Concepts
The NetworkSerializer interface defines a fundamental contract within the server's network layer. It embodies the **Strategy Pattern** for data serialization, establishing a one-way transformation from a high-level game state object into a low-level, transmissible network packet.

This abstraction is critical for decoupling the core game logic from the specifics of the network protocol. Game systems operate on rich, strongly-typed objects (e.g., Player, Entity, WorldChunk), while the network transport requires a standardized, byte-oriented format. The NetworkSerializer acts as the translation bridge between these two domains.

Each implementation of this interface is responsible for serializing exactly one type of game object. These implementations are typically registered in a central registry, which the network dispatcher uses to find the correct serializer for any given object that needs to be sent to a client.

## Lifecycle & Ownership
As a functional interface, NetworkSerializer itself has no lifecycle. The following pertains to the *implementations* of this interface.

-   **Creation:** Implementations are typically defined as stateless singletons or lambdas. They are instantiated once during server bootstrap, as part of the network protocol definition and registration phase.
-   **Scope:** An implementation's lifetime is tied to the network protocol version it belongs to. It persists for the entire duration of the server session.
-   **Destruction:** Implementations are subject to standard garbage collection when the server shuts down and the network registry is cleared.

## Internal State & Concurrency
-   **State:** The contract is inherently stateless. Implementations **must not** contain mutable instance fields. A serializer should be a pure function where the output packet depends solely on the input object. Caching or storing state between calls is a severe anti-pattern.
-   **Thread Safety:** Implementations **must be thread-safe**. The server's network engine may invoke serializers concurrently from multiple I/O worker threads to process outgoing data for different client connections simultaneously. A stateless design is the primary mechanism for achieving this.

## API Surface
The interface exposes a single method, defining its functional contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket(Type object) | Packet | O(N) | Serializes a game object into a network packet. N is the number of fields in the input object. This method must not return null. |

## Integration Patterns

### Standard Usage
A developer's primary interaction is to implement this interface for a custom game object and register it with the network system. The system then invokes it automatically when that object type is dispatched.

```java
// 1. Define the game object
public class PlayerHealthUpdate {
    public final int entityId;
    public final float health;
    // ... constructor
}

// 2. Implement the serializer for that object
public class PlayerHealthUpdateSerializer implements NetworkSerializer<PlayerHealthUpdate, OutgoingPacket> {
    @Override
    public OutgoingPacket toPacket(PlayerHealthUpdate update) {
        OutgoingPacket packet = new OutgoingPacket(PacketIdentifiers.PLAYER_HEALTH);
        packet.getBuffer().writeVarInt(update.entityId);
        packet.getBuffer().writeFloat(update.health);
        return packet;
    }
}

// 3. Register the serializer during server initialization
// (Conceptual example)
PacketRegistry registry = server.getPacketRegistry();
registry.registerSerializer(PlayerHealthUpdate.class, new PlayerHealthUpdateSerializer());
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Implementations:** Storing any form of state, such as a temporary buffer or a counter, as an instance variable within a serializer is forbidden. This will immediately break in a multi-threaded environment, leading to data corruption.
-   **Including Game Logic:** A serializer's responsibility is data transformation only. It should not contain any game logic, such as permission checks or state validation. Such logic must be executed by the calling system *before* the object is queued for serialization.
-   **Complex Object Graphs:** Avoid passing deeply nested or complex object graphs to a serializer. This can lead to performance bottlenecks and excessive memory allocation. Flatten the data into a dedicated, simple Data Transfer Object (DTO) before serialization.

## Data Pipeline
The NetworkSerializer is a key stage in the outbound data pipeline, converting abstract game data into a concrete network representation.

> Flow:
> Game System Logic -> High-Level Object (e.g., PlayerHealthUpdate) -> Network Dispatcher -> **NetworkSerializer** -> Low-Level Packet -> Channel Encoder -> TCP/UDP Socket

