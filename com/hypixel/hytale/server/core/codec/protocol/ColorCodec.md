---
description: Architectural reference for ColorCodec
---

# ColorCodec

**Package:** com.hypixel.hytale.server.core.codec.protocol
**Type:** Utility

## Definition
```java
// Signature
public class ColorCodec implements Codec<Color> {
```

## Architecture & Concepts
The ColorCodec is a specialized component within the Hytale Codec Framework responsible for the serialization and deserialization of the Color data type. It acts as a translation layer, converting the in-memory Java Color object into a standardized string representation suitable for network transmission (BSON) or data storage (JSON), and vice-versa.

Architecturally, this class is an *Adapter* that bridges the high-level game data model (Color) with the low-level wire format. It is not intended for direct use by game logic developers. Instead, the master CodecRegistry automatically discovers and invokes this codec whenever it encounters a Color field during the serialization of a larger network packet or entity.

The core parsing and formatting logic is delegated to the ColorParseUtil helper class. This separation of concerns keeps the ColorCodec focused on its primary role: integrating the Color type into the broader codec system.

### Lifecycle & Ownership
- **Creation:** The ColorCodec is not instantiated directly. It is discovered and instantiated by the central CodecRegistry during server or client bootstrap. The framework scans for classes implementing the Codec interface and registers them against the data type they handle.
- **Scope:** An instance of ColorCodec is effectively a singleton managed by the CodecRegistry. It persists for the entire application session.
- **Destruction:** The object is garbage collected when the CodecRegistry is cleared, typically during application shutdown.

## Internal State & Concurrency
- **State:** The ColorCodec is **stateless**. It contains no member variables and its output is solely dependent on its method inputs. Each method call is an independent, idempotent operation.
- **Thread Safety:** This class is **unconditionally thread-safe**. Its stateless nature allows a single instance to be safely shared and invoked concurrently by multiple network threads without any risk of race conditions or data corruption. This is a critical design requirement for high-performance network serialization.

## API Surface
The public API is the contract required by the Codec interface. These methods are invoked by the parent codec system, not by end-users.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| encode(color, extraInfo) | BsonValue | O(1) | Serializes a Color object into a BsonString (e.g., "#RRGGBB"). |
| decode(bsonValue, extraInfo) | Color | O(1) | Deserializes a BsonString into a Color object. Throws CodecException on invalid format. |
| decodeJson(reader, extraInfo) | Color | O(1) | Performs an optimized deserialization directly from a raw JSON stream. Throws CodecException. |
| toSchema(context) | Schema | O(1) | Generates a formal data schema defining valid color string patterns for validation or tooling. |

## Integration Patterns

### Standard Usage
A developer will almost never interact with the ColorCodec directly. The codec is invoked implicitly by the framework when serializing or deserializing a parent object that contains a Color field.

```java
// Example: A higher-level object containing a Color field
public class PlayerAppearancePacket extends Packet {
    private final String name;
    private final Color favoriteColor; // The framework will find ColorCodec for this
    // ... constructor, etc.
}

// Somewhere in the network layer...
// The framework handles serialization automatically.
// It will see the 'favoriteColor' field and delegate its serialization
// to the registered ColorCodec.
PacketData data = codecRegistry.encode(new PlayerAppearancePacket("Notch", Color.GOLD));

// The same process happens in reverse for decoding.
PlayerAppearancePacket packet = codecRegistry.decode(data);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new ColorCodec()`. The codec system manages the lifecycle and ensures a single, shared instance is used. Direct instantiation bypasses the registry and can lead to unpredictable behavior.
- **Manual Invocation:** Avoid calling `codec.encode(color)` or `codec.decode(value)` manually. The parent serialization process provides critical context via the ExtraInfo parameter and ensures the entire object graph is processed correctly. Manual calls can break the serialization chain.

## Data Pipeline
The ColorCodec sits at a specific point in the data serialization and deserialization pipelines.

> **Serialization Flow:**
> High-Level Packet Object -> CodecRegistry -> **ColorCodec.encode()** -> BsonString -> Full BSON Document -> Network Layer

> **Deserialization Flow:**
> Network Layer -> Full BSON Document -> BsonString -> CodecRegistry -> **ColorCodec.decode()** -> High-Level Color Object

