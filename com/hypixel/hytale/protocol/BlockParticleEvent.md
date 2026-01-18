---
description: Architectural reference for BlockParticleEvent
---

# BlockParticleEvent

**Package:** com.hypixel.hytale.protocol
**Type:** Utility

## Definition
```java
// Signature
public enum BlockParticleEvent {
```

## Architecture & Concepts
BlockParticleEvent is a type-safe enumeration that serves as a fundamental contract within the network protocol. It defines a finite set of physical interactions that can trigger particle effects on or around a block. Its primary architectural role is to translate abstract game events, such as a player walking or a block breaking, into a compact, integer-based representation suitable for network transmission.

This decouples the game logic layer from the low-level rendering and networking layers. The server can simply broadcast an integer ID, and the client can reliably deserialize it back into a specific, named constant. This approach eliminates the use of "magic numbers" in the codebase, improving readability and reducing the risk of desynchronization errors between the client and server.

The inclusion of a static `fromValue` factory method centralizes the deserialization logic, ensuring that any attempt to decode an invalid integer ID results in a controlled `ProtocolException`, which can be handled gracefully by the network layer.

### Lifecycle & Ownership
- **Creation:** Enum constants are instantiated by the Java Virtual Machine during class loading. Each constant (Walk, Run, Break, etc.) is a singleton instance, created only once. The static `VALUES` array is also initialized at this time.
- **Scope:** Application-wide. The enum and its constants persist for the entire lifetime of the application once the class is loaded.
- **Destruction:** The enum and its instances are eligible for garbage collection only when the defining ClassLoader is unloaded, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** Immutable. The internal `value` of each enum constant is a final field, assigned at creation and never modified. The `VALUES` array is a static final reference, and its contents (the enum constants) are also immutable.
- **Thread Safety:** This class is inherently thread-safe. Due to its immutability and the JVM's guarantees for enum initialization, it can be safely accessed and used by multiple threads simultaneously without any external synchronization.

## API Surface
The public contract is designed for serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fromValue(int value) | static BlockParticleEvent | O(1) | Deserializes an integer ID into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |
| getValue() | int | O(1) | Serializes the enum constant into its integer ID for network transmission. |

## Integration Patterns

### Standard Usage
This enum is primarily used by the network packet handling layer to decode incoming data and by the game logic layer to encode outgoing events.

```java
// Example: Deserializing a particle event from a network buffer
int eventId = networkBuffer.readVarInt();
try {
    BlockParticleEvent event = BlockParticleEvent.fromValue(eventId);
    
    // Pass the type-safe event to the particle rendering system
    particleManager.spawnBlockParticles(blockPosition, event);
} catch (ProtocolException e) {
    // Log a warning for malformed packet data
    log.warn("Received invalid BlockParticleEvent ID: " + eventId);
}
```

### Anti-Patterns (Do NOT do this)
- **Using Raw Integers:** Avoid using the integer values directly in game logic. This reintroduces "magic numbers" and defeats the purpose of the type-safe enum.
    ```java
    // BAD: Hardcoding the integer value
    if (eventId == 7) { /* handle break */ }

    // GOOD: Using the named constant
    if (event == BlockParticleEvent.Break) { /* handle break */ }
    ```
- **Ignoring Exceptions:** Failure to catch `ProtocolException` from `fromValue` can lead to unhandled exceptions in the network processing thread, potentially disconnecting the client.

## Data Pipeline
BlockParticleEvent acts as a serialization and deserialization bridge between game logic and the network stream.

> **Outbound Flow (Server):**
> Game Event (Player breaks block) -> Server Logic -> `BlockParticleEvent.Break.getValue()` -> Integer `7` -> Packet Serializer -> Network Stream

> **Inbound Flow (Client):**
> Network Stream -> Packet Deserializer -> Integer `7` -> `BlockParticleEvent.fromValue(7)` -> **BlockParticleEvent.Break** -> Particle System -> Render Effect

