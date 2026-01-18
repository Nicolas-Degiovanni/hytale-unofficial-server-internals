---
description: Architectural reference for MouseInputType
---

# MouseInputType

**Package:** com.hypixel.hytale.protocol
**Type:** Data Type

## Definition
```java
// Signature
public enum MouseInputType {
```

## Architecture & Concepts
The MouseInputType enum is a fundamental data type within the Hytale network protocol. It serves as a type-safe, fixed-size representation of the player's aiming or interaction mode. Its primary function is to categorize and serialize the specific context of a player's look vector, which is critical for server-side validation and game logic.

Instead of transmitting ambiguous "magic numbers" over the network, the engine uses this enum to ensure that both the client and server have a shared, unambiguous understanding of the player's input context. For example, it distinguishes between a player generally looking at a point in space versus specifically targeting a block or an entity.

This component is located in the core protocol package, indicating its role as a foundational piece of the client-server communication contract. Any system that processes player input for network replication will interact with this type.

## Lifecycle & Ownership
- **Creation:** MouseInputType constants are instantiated by the Java Virtual Machine (JVM) during class loading. As an enum, its instances are compile-time constants and are not created dynamically during runtime.
- **Scope:** Application-wide. The enum constants exist for the entire lifetime of the client or server process.
- **Destruction:** The constants are garbage collected only when the JVM itself shuts down. There is no manual memory management required.

## Internal State & Concurrency
- **State:** The MouseInputType enum is **immutable**. Each constant holds a private, final integer value for serialization purposes. The static VALUES array, used for efficient deserialization, is also final and populated only once at class-loading time.
- **Thread Safety:** This class is inherently **thread-safe**. Its immutability guarantees that it can be safely read and passed between any number of threads without locks or synchronization. It is a common data carrier in concurrent networking and game logic systems.

## API Surface
The public contract is designed exclusively for serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the stable integer value for network serialization. |
| fromValue(int value) | MouseInputType | O(1) | Deserializes an integer from a network packet into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum should be used to set or check the state within a larger data structure, typically a network packet representing player actions.

```java
// Example of a system processing a player input packet
void processPlayerInput(PlayerInputPacket packet) {
    if (packet.getMouseInputType() == MouseInputType.LookAtTargetEntity) {
        // Execute logic specific to entity interaction
        handleEntityTargeting(packet.getTargetEntityId());
    } else if (packet.getMouseInputType() == MouseInputType.LookAtTargetBlock) {
        // Execute logic for block interaction
        handleBlockPlacement(packet.getTargetBlockPosition());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Using Raw Integers:** Do not use the underlying integer values for game logic comparisons. This defeats the purpose of type safety and creates brittle code.
  - **BAD:** `if (packet.getMouseInputType().getValue() == 2) { ... }`
  - **GOOD:** `if (packet.getMouseInputType() == MouseInputType.LookAtTargetEntity) { ... }`
- **Relying on Ordinal:** Never use the built-in `ordinal()` method for serialization. The explicit `value` field is provided to ensure the network contract remains stable even if the declaration order of the enum constants changes.
- **Invalid Deserialization:** Do not bypass the `fromValue` method. It contains critical validation logic to prevent protocol corruption from malformed packets.

## Data Pipeline
MouseInputType does not process data itself; it *is* the data. It represents a state field within a larger data flow, typically originating from the client's input system and consumed by the server's game logic.

> **Client-Side Flow:**
> Raw Mouse Input -> Client Input System -> **MouseInputType** (state set in packet) -> PlayerInputPacket -> Network Serializer

> **Server-Side Flow:**
> Network Deserializer -> PlayerInputPacket -> **MouseInputType** (state read from packet) -> Server Game Logic -> World State Update

