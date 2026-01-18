---
description: Architectural reference for ColorAlpha
---

# ColorAlpha

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class ColorAlpha {
```

## Architecture & Concepts

The ColorAlpha class is a fundamental value object within the Hytale network protocol layer. It is not a service or a manager, but rather a low-level data structure designed for the efficient representation and transmission of a 32-bit ARGB (Alpha, Red, Green, Blue) color.

Its primary architectural role is to serve as a **Protocol Data Unit (PDU)** component. The design prioritizes raw performance and memory layout predictability over encapsulation. The class directly maps to a fixed 4-byte block in a network buffer, enabling the protocol serialization engine to perform highly optimized read and write operations without complex parsing logic.

The presence of static constants like FIXED_BLOCK_SIZE and MAX_SIZE signals its use in a rigid, versioned binary protocol where data offsets are pre-calculated. The class acts as a C-style struct, providing a Java-native view into a raw segment of a Netty ByteBuf.

## Lifecycle & Ownership

- **Creation:** ColorAlpha instances are highly ephemeral and created under two primary circumstances:
    1. **Inbound:** The static factory method ColorAlpha.deserialize is invoked by a higher-level packet deserializer when parsing an incoming network stream.
    2. **Outbound:** Game logic instantiates a new ColorAlpha directly (e.g., `new ColorAlpha(a, r, g, b)`) when constructing a packet to be sent over the network.
- **Scope:** The lifetime of a ColorAlpha object is strictly bound to its containing object, typically a parent network packet or a game state component. It is designed to be short-lived.
- **Destruction:** Instances are managed by the Java Garbage Collector. They become eligible for collection as soon as their parent object is no longer referenced. There is no manual destruction or pooling mechanism.

## Internal State & Concurrency

- **State:** The state of ColorAlpha is **fully mutable** through its public byte fields (alpha, red, green, blue). It is a simple data container with no internal logic or caching.

- **Thread Safety:** **This class is not thread-safe.** Its public, mutable fields make it inherently unsafe for concurrent modification. If an instance of ColorAlpha must be shared between threads, access must be controlled by external synchronization mechanisms (e.g., locks or synchronized blocks). Failure to do so will result in race conditions and undefined behavior.

## API Surface

The public contract is designed for direct data manipulation and high-performance serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ColorAlpha(a, r, g, b) | constructor | O(1) | Constructs a new color with the specified component values. |
| deserialize(buf, offset) | static ColorAlpha | O(1) | **Critical Path.** Reads 4 bytes from the buffer at the given offset and constructs a new ColorAlpha instance. Does not modify buffer reader index. |
| serialize(buf) | void | O(1) | **Critical Path.** Writes the 4 color bytes directly into the provided buffer. Advances the buffer writer index. |
| computeSize() | int | O(1) | Returns the constant size of the structure (4 bytes). Used for pre-allocating buffer capacity. |
| validateStructure(buf, offset) | static ValidationResult | O(1) | Verifies if the buffer contains enough readable bytes (at least 4) from the given offset. |
| clone() | ColorAlpha | O(1) | Creates a deep copy of the object. Useful for safely mutating a color without affecting the original. |

## Integration Patterns

### Standard Usage

The class is intended to be used as a transient data holder during network serialization and deserialization.

**Outbound (Serialization):**
```java
// Example: Creating a packet that includes a color
PlayerSkinPacket packet = new PlayerSkinPacket();
packet.hairColor = new ColorAlpha((byte)255, (byte)10, (byte)20, (byte)30);

// The packet's serialize method will later call:
// hairColor.serialize(buffer);
```

**Inbound (Deserialization):**
```java
// Inside a packet's deserialization logic
// The offset is calculated by the parent deserializer
this.hairColor = ColorAlpha.deserialize(buffer, currentOffset);
```

### Anti-Patterns (Do NOT do this)

- **Shared Mutable State:** Do not store a single ColorAlpha instance in a global or shared context where multiple threads can access it. Its mutability makes this pattern extremely dangerous.

    ```java
    // ANTI-PATTERN: A globally accessible color that is modified by multiple threads
    public static ColorAlpha serverStatusColor = new ColorAlpha();

    // Thread 1:
    serverStatusColor.red = 255;

    // Thread 2:
    serverStatusColor.red = 0; // Race condition
    ```

- **Ignoring Fixed Size:** Do not attempt to write variable-length data associated with this object. The entire protocol relies on it consuming exactly 4 bytes.

## Data Pipeline

The ColorAlpha class is a simple link in the network data processing chain. It translates a 4-byte wire format segment into an in-memory object and vice-versa.

> **Inbound Flow:**
> Netty Channel -> ByteBuf -> Packet Deserializer -> **ColorAlpha.deserialize** -> Game Logic

> **Outbound Flow:**
> Game Logic -> `new ColorAlpha()` -> Packet Serializer -> **ColorAlpha.serialize** -> ByteBuf -> Netty Channel

