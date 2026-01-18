---
description: Architectural reference for EntityPart
---

# EntityPart

**Package:** com.hypixel.hytale.protocol
**Type:** Protocol Data Type

## Definition
```java
// Signature
public enum EntityPart {
```

## Architecture & Concepts
The EntityPart enum is a foundational data type within the Hytale network protocol. It serves as a type-safe, compile-time constant for identifying distinct components of a game entity. Its primary function is to eliminate the use of ambiguous "magic numbers" in network packets and game logic, providing a self-documenting and robust mechanism for entity serialization.

This enum acts as a contract between the client and server. When a network packet needs to reference a specific aspect of an entity—such as its main body, a held weapon, or an off-hand item—it uses the integer value defined by an EntityPart constant. This ensures that both endpoints of the connection have an unambiguous understanding of the packet's context, preventing data corruption and desynchronization errors.

It is a critical component for systems that handle entity appearance, inventory, and combat state.

### Lifecycle & Ownership
- **Creation:** All EntityPart constants (Self, Entity, PrimaryItem, SecondaryItem) are instantiated once by the Java Virtual Machine (JVM) when the EntityPart class is loaded. This process is automatic and occurs before any game code can access them.
- **Scope:** Application-scoped. The enum constants exist for the entire lifetime of the application process. They are effectively global, immutable singletons.
- **Destruction:** The constants are garbage collected only when the JVM shuts down. There is no manual memory management or cleanup required.

## Internal State & Concurrency
- **State:** Deeply immutable. Each enum constant holds a private final integer value that is assigned at creation and can never be modified. The static VALUES array is also final, preventing runtime modification.
- **Thread Safety:** Inherently thread-safe. Due to its complete immutability, EntityPart can be safely accessed, passed, and compared across any number of threads without locks or other synchronization primitives. This is a critical property for a high-performance, multi-threaded game engine.

## API Surface
The public contract is minimal, focusing exclusively on serialization and deserialization tasks.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the raw integer value for network serialization. |
| fromValue(int value) | EntityPart | O(1) | **Factory Method.** Deserializes an integer from a network stream into the corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
EntityPart is used during the serialization and deserialization of network packets. The `getValue` method is used for writing to a buffer, and the static `fromValue` method is used for reading.

```java
// Example: Reading an entity update packet from a buffer
int partId = networkBuffer.readVarInt();
EntityPart targetPart = EntityPart.fromValue(partId); // Throws if invalid

// Example: Writing an entity update packet to a buffer
EntityPart sourcePart = EntityPart.PrimaryItem;
networkBuffer.writeVarInt(sourcePart.getValue());
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Exceptions:** The ProtocolException thrown by `fromValue` is a critical security and stability signal. It indicates a malformed, malicious, or out-of-date packet. Swallowing this exception can lead to crashes or undefined behavior. The connection should be terminated.
- **Using Ordinals:** Do not use the built-in `ordinal()` method for serialization. The integer values are explicitly defined in the constructor for protocol stability. Relying on `ordinal()` is brittle and will break the protocol if the declaration order of the enum constants ever changes. Always use `getValue()`.
- **Magic Number Comparisons:** Avoid comparing the integer value directly in game logic. The purpose of the enum is to provide type safety.

    **BAD:**
    `if (somePart.getValue() == 2) { /* logic for primary item */ }`

    **GOOD:**
    `if (somePart == EntityPart.PrimaryItem) { /* logic for primary item */ }`

## Data Pipeline
EntityPart acts as a data transformation point during network I/O, converting between the raw protocol representation (integer) and the type-safe engine representation (enum object).

**Inbound (Deserialization):**
> Flow:
> Network Byte Stream -> Packet Decoder -> **EntityPart.fromValue(intValue)** -> Game Logic Event

**Outbound (Serialization):**
> Flow:
> Game Logic Event -> **somePart.getValue()** -> Packet Serializer -> Network Byte Stream

