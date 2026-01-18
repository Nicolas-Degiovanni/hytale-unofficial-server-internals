---
description: Architectural reference for ColorAlphaCodec
---

# ColorAlphaCodec

**Package:** com.hypixel.hytale.server.core.codec.protocol
**Type:** Utility

## Definition
```java
// Signature
public class ColorAlphaCodec implements Codec<ColorAlpha> {
```

## Architecture & Concepts
The ColorAlphaCodec is a specialized component within the Hytale Serialization Framework. It serves as a low-level translator, responsible for the bidirectional conversion between the server's in-memory representation of a color (the ColorAlpha object) and its serialized string format used in network packets, asset files, and configuration data.

This class is not a standalone service but a pluggable implementation of the Codec interface. The core serialization engine, likely a central CodecRegistry, discovers and delegates to this specific implementation whenever it encounters a ColorAlpha type during a serialization or deserialization operation.

Its primary function is to enforce a strict, well-defined contract for how color data is represented as a string, supporting multiple common formats like hex and RGBA. The inclusion of the toSchema method indicates a deep integration with a schema-driven validation system, allowing the engine to validate data formats *before* attempting to decode them, thus ensuring data integrity and preventing runtime errors.

### Lifecycle & Ownership
- **Creation:** Instances of ColorAlphaCodec are not intended for manual creation. They are instantiated by the Hytale serialization framework during its bootstrap phase. The framework scans for and registers all Codec implementations, making them available for system-wide use.
- **Scope:** As a stateless utility, a single instance of ColorAlphaCodec is created and shared for the entire lifetime of the server or client process. It persists from application start to shutdown.
- **Destruction:** The object is eligible for garbage collection only when the application terminates and its ClassLoader is unloaded. There is no manual destruction logic.

## Internal State & Concurrency
- **State:** The ColorAlphaCodec is **stateless**. It contains no member fields and its output is exclusively determined by its input arguments. Each method call is an independent, atomic operation.
- **Thread Safety:** This class is **inherently thread-safe**. Due to its stateless nature, a single shared instance can be safely and concurrently invoked by multiple threads (e.g., network I/O workers, asset loaders) without any risk of race conditions or data corruption. No external locking or synchronization is required.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| encode(ColorAlpha, ExtraInfo) | BsonValue | O(1) | Serializes a ColorAlpha object into a BsonString containing a hex color string (#RRGGBBAA). |
| decode(BsonValue, ExtraInfo) | ColorAlpha | O(N) | Deserializes a BsonString into a ColorAlpha object. Throws CodecException if the string format is invalid. N is the length of the string. |
| decodeJson(RawJsonReader, ExtraInfo) | ColorAlpha | O(N) | Deserializes a color string directly from a raw JSON stream. Throws CodecException on format mismatch or IOException on stream errors. |
| toSchema(SchemaContext) | Schema | O(1) | Generates a data schema defining the multiple valid string patterns for a color. Used for upstream data validation. |

## Integration Patterns

### Standard Usage
Developers should **never** interact with this class directly. The serialization framework uses it implicitly. You operate on higher-level objects, and the framework ensures the correct codec is invoked.

```java
// Define an object containing a color field
public class UILayout {
    public String name;
    public ColorAlpha backgroundColor;
}

// The framework automatically finds and uses ColorAlphaCodec
// when encoding or decoding the UILayout object.
UILayout layout = new UILayout();
layout.name = "main_menu";
layout.backgroundColor = new ColorAlpha(0, 0, 0, 128); // Black, 50% transparent

// This operation will implicitly invoke ColorAlphaCodec.encode()
// for the backgroundColor field.
BsonValue bson = CodecRegistry.getShared().encode(layout);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new ColorAlphaCodec()`. The framework manages a shared singleton instance. Direct creation bypasses the registry and can lead to unpredictable behavior.
- **Manual Invocation:** Do not call the encode or decode methods directly. Rely on the central CodecRegistry or equivalent serialization entry point to process your data structures. Manually calling the codec breaks the abstraction and makes the code brittle.
- **Exception-based Logic:** Do not use a try-catch block around a decode call to validate a color string. The `toSchema` method provides the correct patterns for upstream validation before a decode operation is ever attempted. Relying on CodecException for validation is inefficient and poor practice.

## Data Pipeline
The ColorAlphaCodec sits at the lowest level of the data transformation pipeline, acting as the final bridge between the object world and the serialized world.

**Serialization (Encode)**
> Flow:
> Game Logic creates `ColorAlpha` Object -> Serialization Engine -> **ColorAlphaCodec.encode()** -> `BsonString` (`"#RRGGBBAA"`) -> Network Packet or JSON/BSON File

**Deserialization (Decode)**
> Flow:
> Network Packet or JSON/BSON File -> Parser creates `BsonString` -> Schema Validator (Optional) -> Serialization Engine -> **ColorAlphaCodec.decode()** -> `ColorAlpha` Object -> Game Logic<ctrl63>

