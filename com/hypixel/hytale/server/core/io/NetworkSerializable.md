---
description: Architectural reference for NetworkSerializable
---

# NetworkSerializable

**Package:** com.hypixel.hytale.server.core.io
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface NetworkSerializable<Packet> {
```

## Architecture & Concepts
The NetworkSerializable interface defines a fundamental contract for data serialization within the Hytale networking layer. It establishes a standardized protocol for any game state object that must be converted into a network-transmissible format, generically referred to as a Packet.

This interface acts as a crucial decoupling mechanism. It allows high-level game logic objects, such as EntityState or PlayerInventory, to define their own network representation without needing any knowledge of the underlying network transport, buffers, or protocols. Conversely, the low-level network sender components can operate on any object that fulfills the NetworkSerializable contract, treating them as opaque data sources to be converted into packets and sent over the wire.

This pattern is central to the engine's data-oriented networking design, promoting a clear separation of concerns between game state management and network communication.

### Lifecycle & Ownership
As an interface, NetworkSerializable does not have a lifecycle of its own. It is a compile-time contract, not a runtime object. The lifecycle and ownership concerns apply to the *classes that implement* this interface.

- **Creation:** Implementing objects are created according to their own specific logic (e.g., a PlayerState object is created when a player joins).
- **Scope:** The scope of the implementing object varies. Some may be short-lived, representing a single event, while others may persist for an entire session.
- **Destruction:** The implementing object is responsible for its own cleanup and garbage collection.

## Internal State & Concurrency
- **State:** An interface has no internal state. All state is contained within the implementing class.
- **Thread Safety:** This interface makes no guarantees about thread safety. The responsibility for ensuring safe concurrent access lies entirely with the implementing class.

**WARNING:** Implementations of the toPacket method must be thread-safe if the object can be accessed by multiple threads, such as the main game thread and a dedicated network thread. It is highly recommended that implementations are either immutable or employ appropriate synchronization mechanisms to prevent data corruption during serialization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | Packet | O(N) | Serializes the object's current state into a network-ready Packet. The complexity is proportional to the number of fields being serialized. |

## Integration Patterns

### Standard Usage
The primary pattern is to implement this interface on data-transfer objects (DTOs) or state-holding objects that need to be synchronized between the client and server.

```java
// 1. A game object implements the interface
public class PlayerPositionUpdate implements NetworkSerializable<PositionPacket> {
    private Vector3 position;
    private Vector3 velocity;

    @Override
    public PositionPacket toPacket() {
        // Convert internal state to a specific packet type
        PositionPacket packet = new PositionPacket();
        packet.setX(this.position.x);
        packet.setY(this.position.y);
        // ... and so on
        return packet;
    }
}

// 2. The network system uses the contract to send the object
PlayerPositionUpdate update = player.getPositionUpdate();
networkManager.send(update.toPacket());
```

### Anti-Patterns (Do NOT do this)
- **Heavy Computation:** Do not perform blocking I/O, complex calculations, or database lookups within the toPacket method. This method is often called on a performance-critical network thread and must return quickly.
- **State Mutation:** The toPacket method should be a pure function with no side effects. It must not modify the internal state of the object it is serializing.
- **Implementing on God Objects:** Avoid implementing NetworkSerializable on massive, monolithic classes. This leads to bloated packets and tightly couples the entire object's state to the network protocol. Instead, create smaller, dedicated DTOs that implement the interface.

## Data Pipeline
This interface is the first step in the outbound data serialization pipeline. It transforms a high-level game object into a low-level, structured packet.

> Flow:
> Game State Object -> **toPacket()** -> Raw Packet Object -> Packet Encoder -> Network Buffer -> Socket

