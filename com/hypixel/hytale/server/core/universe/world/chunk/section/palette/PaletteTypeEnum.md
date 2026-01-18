---
description: Architectural reference for PaletteTypeEnum
---

# PaletteTypeEnum

**Package:** com.hypixel.hytale.server.core.universe.world.chunk.section.palette
**Type:** Utility

## Definition
```java
// Signature
public enum PaletteTypeEnum {
```

## Architecture & Concepts
The PaletteTypeEnum serves as a high-performance, type-safe factory for creating ISectionPalette instances. In Hytale's world data model, a chunk section uses a palette to compress block data. Instead of storing the full block ID for every position, the section stores a small index into a local palette. This enum is the central dispatcher that maps a serialized palette type identifier (a single byte) to the correct, memory-optimized palette implementation.

Its primary architectural role is to decouple the world serialization and chunk management systems from the concrete palette implementations (e.g., HalfByteSectionPalette, ByteSectionPalette). This allows the engine to select the most space-efficient palette strategy for a given chunk section without requiring the calling code to contain complex conditional logic.

The enum leverages Java's Supplier functional interface to defer the instantiation of a palette until it is explicitly requested, acting as a registry of constructors.

### Lifecycle & Ownership
- **Creation:** All enum constants (EMPTY, HALF_BYTE, BYTE, SHORT) are instantiated by the Java Virtual Machine during static class initialization. This process is guaranteed to happen only once.
- **Scope:** The enum instances are static and persist for the entire lifetime of the server application. They are effectively global singletons.
- **Destruction:** The enum and its instances are garbage collected only when the server process terminates and its classloader is unloaded. There is no manual destruction mechanism.

## Internal State & Concurrency
- **State:** The PaletteTypeEnum is **immutable**. All its fields, including the internal values array, are final and are initialized once during static setup. It holds no mutable instance state.
- **Thread Safety:** This class is unconditionally thread-safe. Its immutability and the nature of Java enums guarantee that it can be safely accessed by any number of threads concurrently without external synchronization. The get method is a non-blocking, atomic array lookup.

## API Surface
The public API provides the necessary tools to resolve a palette type and acquire a constructor for its corresponding implementation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(byte paletteId) | PaletteTypeEnum | O(1) | **Static Factory Method.** Resolves a PaletteTypeEnum constant from its serialized byte identifier. Throws ArrayIndexOutOfBoundsException if the ID is invalid. |
| getPaletteType() | PaletteType | O(1) | Returns the underlying protocol-level enum type associated with this palette. |
| getConstructor() | Supplier | O(1) | Returns the constructor, encapsulated in a Supplier, for the associated ISectionPalette implementation. |
| getPaletteId() | byte | O(1) | Returns the unique byte identifier used for network and disk serialization. |

## Integration Patterns

### Standard Usage
The canonical use case is during chunk deserialization. The system reads a palette ID from a data stream, uses this enum to find the correct type, and then invokes the constructor to create the appropriate palette object.

```java
// context: Reading chunk data from a network buffer or file
byte paletteId = buffer.readByte();

// Use the enum as a factory to get the correct palette type
PaletteTypeEnum paletteInfo = PaletteTypeEnum.get(paletteId);

// Use the supplier to instantiate the correct palette implementation
ISectionPalette sectionPalette = paletteInfo.getConstructor().get();

// The sectionPalette is now ready to be populated with data
sectionPalette.read(buffer);
```

### Anti-Patterns (Do NOT do this)
- **Invalid ID Handling:** Do not call PaletteTypeEnum.get(id) without validating the input ID. An invalid ID, potentially from corrupted data or a malicious client, will cause an unhandled ArrayIndexOutOfBoundsException, crashing the thread responsible for chunk loading. Always bounds-check the ID first.
- **Relying on Ordinal:** Do not use the built-in enum ordinal() method for serialization. The code correctly uses an explicitly assigned paletteId derived from the protocol definition. Relying on ordinal() is fragile and will break data compatibility if the enum declaration order is ever changed.

## Data Pipeline
This enum acts as a critical dispatch point in the chunk data loading pipeline, translating a low-level data identifier into a high-level object factory.

> Flow:
> Network Buffer or File Stream -> Chunk Deserializer reads `paletteId` -> **PaletteTypeEnum.get()** -> `getConstructor().get()` -> Instantiated ISectionPalette -> Populated from Stream

