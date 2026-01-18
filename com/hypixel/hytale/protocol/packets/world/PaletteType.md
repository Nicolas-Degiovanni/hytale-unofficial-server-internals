---
description: Architectural reference for PaletteType
---

# PaletteType

**Package:** com.hypixel.hytale.protocol.packets.world
**Type:** Utility

## Definition
```java
// Signature
public enum PaletteType {
```

## Architecture & Concepts
The PaletteType enum is a critical component of the world data serialization and deserialization protocol. It acts as a type-safe discriminator that defines the storage format and bit-depth for a chunk's block palette. This is a fundamental compression strategy used to reduce the network bandwidth and disk space required for world data.

In Hytale's voxel engine, a chunk is divided into sections, and each section can use a palette to map a small, local ID to a global, comprehensive block state ID. PaletteType specifies how these local IDs are encoded.

- **Empty (0):** Represents a chunk section composed of a single block type (e.g., all air). No palette data is needed.
- **HalfByte (1):** Uses 4 bits per block, allowing for a palette of up to 16 unique block types.
- **Byte (2):** Uses 8 bits per block, allowing for a palette of up to 256 unique block types.
- **Short (3):** Uses 16 bits per block, effectively a direct mapping to global block state IDs, bypassing the palette system for highly complex chunk sections.

This enum is the primary mechanism by which the network protocol decoder determines how to read the subsequent block data stream for a given chunk section.

### Lifecycle & Ownership
- **Creation:** PaletteType instances are constants, instantiated by the Java Virtual Machine during class loading. User code does not and cannot create instances of this enum.
- **Scope:** Application-wide. The enum constants exist for the entire lifetime of the application.
- **Destruction:** The instances are reclaimed by the garbage collector only when the application's classloader is unloaded, which typically occurs at shutdown.

## Internal State & Concurrency
- **State:** **Immutable**. The internal integer value and the static array of values are final and established at class-loading time. The state of a PaletteType instance can never be changed.
- **Thread Safety:** **Fully thread-safe**. As an immutable, JVM-managed construct, PaletteType can be safely accessed and used from any thread without synchronization.

## API Surface
The public contract is minimal, focusing on conversion between the enum constant and its underlying network protocol identifier.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer value used for serialization in the network protocol. |
| fromValue(int value) | PaletteType | O(1) | A static factory method to deserialize an integer from the network stream into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is almost exclusively used within the network protocol decoding layer to branch logic based on the palette format received from the server or read from disk.

```java
// Example within a hypothetical packet decoder
int formatId = byteBuf.readUnsignedByte();
PaletteType paletteType = PaletteType.fromValue(formatId);

switch (paletteType) {
    case Empty:
        // Logic for single-block-type chunks
        break;
    case HalfByte:
        // Logic to read 4-bit palette indices
        break;
    case Byte:
        // Logic to read 8-bit palette indices
        break;
    // ... and so on
}
```

### Anti-Patterns (Do NOT do this)
- **Using Magic Numbers:** Do not use raw integers (0, 1, 2, 3) in your application logic to represent the palette type. This defeats the purpose of type safety and creates brittle code. Always use the enum constants like PaletteType.Byte.
- **Invalid Deserialization:** Do not manually check if a value is valid before calling fromValue. The method is designed to perform this validation and throw a standardized ProtocolException, which should be caught by the higher-level packet processing logic.

## Data Pipeline
PaletteType functions as a control signal within the data-in pipeline, directing the flow of deserialization for world data.

> Flow:
> Network Byte Stream -> Protocol Deserializer reads integer ID -> **PaletteType.fromValue()** -> Conditional Deserialization Logic -> Populated Chunk Data Structure

