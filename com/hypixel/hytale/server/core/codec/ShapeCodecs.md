---
description: Architectural reference for ShapeCodecs
---

# ShapeCodecs

**Package:** com.hypixel.hytale.server.core.codec
**Type:** Utility

## Definition
```java
// Signature
public class ShapeCodecs {
```

## Architecture & Concepts
The ShapeCodecs class is a static registry that provides the core serialization and deserialization logic for all geometric **Shape** subtypes within the engine. It acts as a centralized, authoritative source for converting shape objects to and from persistent data formats, such as world files or network packets.

This class is a critical component of the data persistence and networking layers. Its primary architectural role is to enable **polymorphic serialization** of shapes. The master codec, **SHAPE**, is a **CodecMapCodec**, which functions as a dispatcher. When deserializing data, it first reads a type identifier string (e.g., "Box", "Cylinder") and then delegates the remainder of the process to the corresponding specialized **BuilderCodec** (e.g., **BOX**, **CYLINDER**). This design allows diverse shape types to be stored and transmitted interchangeably within any system that requires a generic **Shape** reference, such as entity definitions, trigger volumes, or physics colliders.

The implementation relies entirely on the engine's generic **Codec** framework, composing specialized codecs from framework primitives rather than implementing custom parsing logic.

### Lifecycle & Ownership
- **Creation:** The codecs defined in this class are static final fields. They are instantiated and configured by the JVM class loader precisely once, when the **ShapeCodecs** class is first referenced. The static initializer block populates the master **SHAPE** registry during this one-time event.
- **Scope:** The codecs are global, application-scoped singletons. They persist for the entire lifetime of the server or client process.
- **Destruction:** The objects are reclaimed by the JVM only when the application shuts down and its class loader is garbage collected. There is no manual destruction or cleanup process.

## Internal State & Concurrency
- **State:** The codecs are effectively **immutable** after class initialization. The internal map of the **SHAPE** codec is populated by the static initializer and is not modified further during runtime. The individual **BuilderCodec** instances are stateless by design; they process input data without retaining any information between calls.
- **Thread Safety:** All provided codecs are **thread-safe**. Their immutable and stateless nature guarantees that they can be used concurrently by multiple threads—for example, by worker threads processing world chunks or handling network I/O—without requiring external locking or synchronization. This is a fundamental requirement for performance in the server's multi-threaded architecture.

## API Surface
The public contract of this class consists solely of its static final codec fields. The primary entry point for most systems is the polymorphic **SHAPE** codec.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SHAPE | CodecMapCodec<Shape> | O(1) | The master polymorphic codec. Dispatches to a specific shape codec based on a type key in the data. |
| BOX | BuilderCodec<Box> | O(N) | Serializes and deserializes a **Box** shape, defined by Min and Max vectors. |
| ELLIPSOID | BuilderCodec<Ellipsoid> | O(N) | Serializes and deserializes an **Ellipsoid** shape, defined by its three radii. |
| CYLINDER | BuilderCodec<Cylinder> | O(N) | Serializes and deserializes a **Cylinder** shape, defined by its height and radii. |
| ORIGIN_SHAPE | BuilderCodec<OriginShape> | O(N) | Serializes a composite shape with an origin offset. Recursively uses the **SHAPE** codec for its child. |

## Integration Patterns

### Standard Usage
Systems should never reference the specific codecs like **BOX** or **CYLINDER** directly unless they are guaranteed to only ever handle that one type. The standard pattern is to use the master **SHAPE** codec to support any valid geometric shape.

This is typically done when defining another codec for a component that contains a shape.

```java
// Example: Codec for a component containing a generic shape
public static final BuilderCodec<TriggerVolume> TRIGGER_VOLUME_CODEC = BuilderCodec.builder(TriggerVolume.class, TriggerVolume::new)
    .addField(
        new KeyedCodec<>("Bounds", ShapeCodecs.SHAPE), // Correct usage
        (volume, shape) -> volume.setBounds(shape),
        TriggerVolume::getBounds
    )
    .build();
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** This is a static utility class and cannot be instantiated. Attempting to use `new ShapeCodecs()` will fail.
- **Runtime Registration:** The **SHAPE** codec's `register` method is public, but it must **never** be called outside of the static initializer block. Modifying the codec map at runtime is not thread-safe and will lead to catastrophic, non-deterministic behavior across the application. All shape codecs must be registered at class-load time.

## Data Pipeline
The following illustrates the data flow during a typical deserialization operation using the master **SHAPE** codec.

> Flow:
> Serialized Data Stream (e.g., from a file or network) -> Engine Codec Processor -> **ShapeCodecs.SHAPE** reads type identifier ("Box") -> Dispatches to **ShapeCodecs.BOX** -> **ShapeCodecs.BOX** reads fields ("Min", "Max") and constructs a new **Box** instance -> The fully-formed **Box** object is returned to the Engine Codec Processor.

