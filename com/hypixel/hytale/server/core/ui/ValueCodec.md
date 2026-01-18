---
description: Architectural reference for ValueCodec
---

# ValueCodec<T>

**Package:** com.hypixel.hytale.server.core.ui
**Type:** Utility

## Definition
```java
// Signature
public class ValueCodec<T> implements Codec<Value<T>> {
```

## Architecture & Concepts
The ValueCodec is a specialized serialization component within Hytale's data codec framework. Its primary function is to translate a **Value<T>** object, a common wrapper in the UI system, into its BSON representation. This class is not a general-purpose codec; it is a key part of the architecture that enables Hytale's UI definitions to be both flexible and efficient.

The core architectural pattern implemented by ValueCodec is **dual-mode encoding**. It can serialize a Value<T> object in one of two ways:

1.  **As a Direct Value:** If the Value<T> object contains a concrete, non-null value (e.g., the string "Hello"), the ValueCodec delegates the serialization task to an inner, type-specific codec (e.g., Codec.STRING).
2.  **As a Reference:** If the Value<T> object's internal value is null, it is treated as a symbolic link or reference to a value defined elsewhere (e.g., in a shared style document). In this case, the ValueCodec produces a BSON document containing the path to the target document and the name of the value within it.

This dual-mode capability is critical for creating reusable and themeable UI components, allowing designers to define styles in a central location and have UI elements reference them by path.

**WARNING:** This is a strictly **one-way, encode-only** codec. The decode method is not implemented and will throw an UnsupportedOperationException. This is a deliberate design choice, enforcing a separation of concerns where other systems are responsible for resolving references and deserializing data.

## Lifecycle & Ownership
-   **Creation:** The ValueCodec is not intended for direct instantiation by consumers. The class provides a set of pre-configured, public static final instances for common types (e.g., STRING, INTEGER, LOCALIZABLE_STRING). These instances are created once at class-loading time.
-   **Scope:** The static instances are application-scoped singletons. They persist for the entire lifetime of the server process.
-   **Destruction:** The objects are managed by the Java Virtual Machine and are garbage collected during application shutdown. No manual cleanup is necessary.

## Internal State & Concurrency
-   **State:** The ValueCodec is **immutable and stateless**. Its single instance field, a reference to the inner codec, is final and set during construction. The public static instances are constants.
-   **Thread Safety:** This class is inherently **thread-safe**. Because it holds no mutable state, its encode method can be safely invoked by multiple threads concurrently without external synchronization. The encoding process is a pure function of its inputs.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| encode(Value<T> r, ExtraInfo) | BsonValue | O(N) | Encodes a Value object into BSON. Complexity is dependent on the inner codec. Throws NullPointerException if the inner codec is null and a direct value is provided. |
| decode(BsonValue, ExtraInfo) | Value<T> | N/A | **Unsupported.** Throws UnsupportedOperationException upon invocation. This method must not be called. |
| toSchema(SchemaContext) | Schema | O(N) | Delegates schema generation to the inner codec. |

## Integration Patterns

### Standard Usage
Developers should exclusively use the provided static instances when a codec for a Value<T> is required by another system, such as a component serializer.

```java
// A system needs to encode a UI property that is a Value<String>.
// This value might be a direct string or a reference to a style guide.

Value<String> directValue = Value.of("Click Me");
Value<String> referenceValue = Value.ofReference("global_styles.bson", "primary_button_text");

// Use the predefined static codec for strings.
BsonValue encodedDirect = ValueCodec.STRING.encode(directValue, extraInfo);
BsonValue encodedReference = ValueCodec.STRING.encode(referenceValue, extraInfo);
```

### Anti-Patterns (Do NOT do this)
-   **Attempting to Decode:** Do not call the decode method. It is not implemented and will crash the calling thread. Data resolution and deserialization are handled by a different part of the engine.
-   **Direct Instantiation:** The constructor is not public. Do not attempt to create new instances of ValueCodec using reflection. This violates the design contract and bypasses the intended singleton pattern for common types.
-   **Misusing REFERENCE_ONLY:** The `ValueCodec.REFERENCE_ONLY` instance is a special-purpose codec with a null inner codec. It can *only* be used to encode reference-based Value objects. Attempting to encode a Value containing a direct, non-null value with it will result in a NullPointerException.

## Data Pipeline
The ValueCodec acts as a terminal step in the object-to-BSON serialization pipeline for UI Value objects.

> Flow:
> In-Memory **Value<T>** Object -> **ValueCodec.encode()** -> Serialized **BsonValue** -> Network Buffer / Disk Storage

