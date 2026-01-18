---
description: Architectural reference for VariantRotation
---

# VariantRotation

**Package:** com.hypixel.hytale.protocol
**Type:** Utility

## Definition
```java
// Signature
public enum VariantRotation {
```

## Architecture & Concepts
The **VariantRotation** enum is a foundational data type within the Hytale protocol layer. Its primary function is to provide a type-safe, human-readable representation for the geometric orientation of a block or entity variant. It serves as a critical bridge between high-level game logic and the low-level binary data stream sent over the network.

Each constant, such as **Wall** or **Pipe**, maps a semantic orientation to a compact integer value. This design is highly efficient for network serialization, minimizing packet size. The inclusion of a static factory method, **fromValue**, centralizes the deserialization logic and enforces protocol correctness by rejecting invalid integer values with a **ProtocolException**. This makes the system resilient to corrupted or malformed data packets.

This enum is not a service or a manager; it is a static data contract that defines a fixed set of possible states.

### Lifecycle & Ownership
- **Creation:** Instances are created and managed exclusively by the Java Virtual Machine during class loading. As an enum, its constants are singleton instances that are initialized once.
- **Scope:** Application-wide. The **VariantRotation** constants are available for the entire lifetime of the application.
- **Destruction:** The enum and its constants are garbage collected when the application's ClassLoader is unloaded, typically upon JVM shutdown.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant holds a final, private integer value that cannot be changed after initialization. The static **VALUES** array is also final.
- **Thread Safety:** **VariantRotation** is inherently thread-safe. Its immutability and the JVM's guarantees for enum initialization ensure that it can be safely accessed and read from any thread without requiring external synchronization or locks.

## API Surface
The public contract is minimal, focusing on conversion between the enum constant and its underlying integer representation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer value defined in the protocol for this rotation. |
| fromValue(int value) | VariantRotation | O(1) | **Static**. Deserializes an integer into the corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used during network packet serialization and deserialization. Game logic should always operate on the enum constants, converting to or from integers only at the protocol boundary.

```java
// Example: Deserializing a block update from a network buffer
int rotationValue = buffer.readVarInt();
VariantRotation rotation = VariantRotation.fromValue(rotationValue);

// Now use the type-safe enum in game logic
world.getBlock(pos).setRotation(rotation);
```

### Anti-Patterns (Do NOT do this)
- **Using Magic Numbers:** Avoid using raw integer values (0, 1, 2, etc.) directly in game logic. This creates brittle code that is difficult to read and maintain. Always use the named constants like **VariantRotation.Wall**.
- **Relying on Ordinal:** Do not use the built-in `ordinal()` method for serialization. The protocol contract is based on the explicit `value` field. The ordinal can change if the declaration order of the enum constants is modified, which would break network compatibility.

## Data Pipeline
**VariantRotation** is a data model, not an active component. It represents data that flows through the system.

**Serialization (Client to Server)**
> Game Logic (e.g., Player places a block) -> Block State with **VariantRotation.Wall** -> **getValue()** -> Integer `1` -> Network Packet Encoder -> TCP Stream

**Deserialization (Server to Client)**
> TCP Stream -> Network Packet Decoder -> Integer `1` -> **fromValue(1)** -> **VariantRotation.Wall** -> Block State Update -> World Renderer

