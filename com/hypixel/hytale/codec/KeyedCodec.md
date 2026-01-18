---
description: Architectural reference for KeyedCodec
---

# KeyedCodec

**Package:** com.hypixel.hytale.codec
**Type:** Transient

## Definition
```java
// Signature
public class KeyedCodec<T> {
```

## Architecture & Concepts

The KeyedCodec is a fundamental component in the Hytale serialization framework. It does not perform serialization itself, but rather acts as a **binding** between a string key and a delegate Codec responsible for a specific data type T. Its primary role is to orchestrate the retrieval or placement of a typed value within a BSON key-value structure, specifically a BsonDocument.

Conceptually, a KeyedCodec represents a single, named field within a larger data contract. For example, in serializing a Player object, one KeyedCodec might handle the "Name" field (binding the key "Name" to a StringCodec), while another handles the "Health" field (binding "Health" to an IntCodec).

A critical architectural feature is the enforcement of a key naming convention. By default, keys must begin with an uppercase character. This convention is enforced at construction time and suggests that these keys map directly to high-level, contract-defined fields, similar to properties in a class.

The system also provides a mechanism for data inheritance via the `getAndInherit` method. This specialized path allows a KeyedCodec to delegate to an `InheritCodec`, enabling complex data templating where an object can inherit and override values from a parent object. This is essential for systems like entity prefabs or item definitions.

Error handling is managed through the `ExtraInfo` context object, which is passed through all operations. The KeyedCodec contributes to this context by pushing its key onto the stack before delegation, ensuring that any downstream `CodecException` contains a precise path to the source of the error within the nested BSON structure.

### Lifecycle & Ownership
- **Creation:** A KeyedCodec is typically instantiated once during application startup as part of a larger, composite codec definition (e.g., a RecordCodec). It is configured with its key, a delegate codec, and a `required` flag.
- **Scope:** The object is designed to be a reusable, stateless configuration object. Its lifetime is tied to the composite codec that owns it. In most cases, these are defined as static final fields and persist for the entire application lifetime.
- **Destruction:** Managed by the Java garbage collector when its owning codec is unloaded. No explicit cleanup is required.

## Internal State & Concurrency
- **State:** The internal state of a KeyedCodec (its key, delegate codec, and required flag) is **effectively immutable**. All fields are final and are set only once within the constructor.
- **Thread Safety:** The KeyedCodec class is **thread-safe**. Its immutable nature ensures that multiple threads can safely call its methods concurrently without risk of data corruption. Thread safety of a complete serialization operation depends on the thread safety of the BsonDocument being manipulated and the delegate Codec.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(document, extraInfo) | Optional<T> | O(1) | Decodes the value for the configured key. Returns an empty Optional if the key is absent or the value is null. |
| getNow(document, extraInfo) | T | O(1) | Decodes the value for the configured key. Throws BsonSerializationException if the key is absent. |
| getOrNull(document, extraInfo) | T | O(1) | Decodes the value for the configured key. Returns null if the key is absent or the value is null. |
| getOrDefault(document, extraInfo, def) | T | O(1) | Decodes the value for the configured key. Returns the provided default value if the key is absent. |
| put(document, t, extraInfo) | void | O(1) | Encodes the provided object and inserts the resulting BsonValue into the document under the configured key. |
| getAndInherit(document, parent, extraInfo) | Optional<T> | O(1) | Performs a specialized decode operation that supports data inheritance from a parent object. |
| isRequired() | boolean | O(1) | Returns true if the key is considered mandatory during decoding. |

## Integration Patterns

### Standard Usage

A KeyedCodec is almost never used in isolation. It is a building block for creating codecs for complex objects. The standard pattern is to define multiple KeyedCodec instances as static fields and compose them into a higher-level codec, such as a RecordCodec.

```java
// Example of defining a codec for a simple data object
public class PlayerStats {
    // Define codecs for individual fields
    public static final KeyedCodec<String> NAME = new KeyedCodec<>("Name", Codecs.STRING, true);
    public static final KeyedCodec<Integer> HEALTH = new KeyedCodec<>("Health", Codecs.INTEGER, true);
    public static final KeyedCodec<Integer> MANA = new KeyedCodec<>("Mana", Codecs.INTEGER, false);

    // These would then be used by a larger RecordCodec to (de)serialize a PlayerStats object.
    // BsonDocument doc = ...;
    // String playerName = PlayerStats.NAME.getNow(doc, extraInfo);
    // int playerHealth = PlayerStats.HEALTH.getNow(doc, extraInfo);
}
```

### Anti-Patterns (Do NOT do this)
- **Repetitive Instantiation:** Do not create new KeyedCodec instances within loops or frequently called methods. They are lightweight but designed to be defined once and reused. Define them as static final constants.
- **Ignoring ExtraInfo:** Avoid using the deprecated methods that accept no ExtraInfo parameter. These methods provide significantly less context upon failure, making debugging serialization errors extremely difficult.
- **Contract Violation:** Do not use keys that begin with a lowercase letter. The constructor will throw an IllegalArgumentException. This convention is in place to ensure consistency across all data definitions.

## Data Pipeline

The KeyedCodec acts as a dispatcher in the data pipeline, directing data to and from a BsonDocument based on its configured key.

**Decoding Flow:**
> BsonDocument -> KeyedCodec.get() -> BsonDocument lookup by key -> BsonValue -> Delegate Codec.decode() -> Java Object T

**Encoding Flow:**
> Java Object T -> KeyedCodec.put() -> Delegate Codec.encode() -> BsonValue -> BsonDocument insertion with key

