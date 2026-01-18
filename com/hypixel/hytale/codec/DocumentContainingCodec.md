---
description: Architectural reference for DocumentContainingCodec
---

# DocumentContainingCodec

**Package:** com.hypixel.hytale.codec
**Type:** Utility / Decorator

## Definition
```java
// Signature
@Deprecated
public class DocumentContainingCodec<T> extends BsonFunctionCodec<T> {
```

## Architecture & Concepts
The DocumentContainingCodec is a specialized decorator for BSON serialization that enables forward compatibility by preserving unknown fields. It wraps a primary BuilderCodec, which handles the explicitly defined, strongly-typed fields of a data model.

The core architectural purpose of this class is to solve the "schema evolution" problem. When a BSON document is decoded, this codec first allows the wrapped BuilderCodec to process all known fields. It then isolates any remaining fields—those not defined in the BuilderCodec—and injects them as a single BsonDocument into a designated "catch-all" field on the target Java object.

Conversely, during encoding, it retrieves this "catch-all" BsonDocument from the Java object and merges its contents back into the main BSON structure alongside the explicitly defined fields. This ensures that data added in a newer version of the application is not lost when an older version reads and then re-writes the data.

**WARNING:** This class is marked as **Deprecated**. Its use in new development is strictly prohibited. It exists for legacy compatibility and should be replaced by more modern, robust schema management strategies.

### Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level codec factory or registry during the construction of a serialization pipeline for a specific data type. It is not intended to be created directly by application logic.
- **Scope:** Transient. The lifecycle of a DocumentContainingCodec instance is bound to the parent codec that composes it. It does not persist beyond the scope of a single serialization or deserialization operation.
- **Destruction:** The object is eligible for garbage collection as soon as the parent codec is no longer referenced. It requires no explicit resource management or cleanup.

## Internal State & Concurrency
- **State:** Stateless. This codec maintains no internal state between invocations. Its behavior is determined entirely by the functions and wrapped codec provided at construction time.
- **Thread Safety:** Not inherently thread-safe. The concurrency safety of this codec is entirely dependent on the thread-safety guarantees of the wrapped BuilderCodec and the getter/setter functions supplied to its constructor. It is designed to be used within a single-threaded serialization context. Unsynchronized, concurrent access will lead to unpredictable behavior and data corruption.

## API Surface
The public API consists solely of the constructor. The primary functionality is exposed through the Codec interface and is invoked by the BSON serialization engine, not by application code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DocumentContainingCodec(codec, setter, getter) | Constructor | O(1) | Constructs the decorator codec. Requires a primary codec for known fields, a setter to inject unknown fields, and a getter to extract them. |

## Integration Patterns

### Standard Usage
The following example demonstrates the *historical* usage pattern. This pattern is now considered an anti-pattern for new code.

```java
// Within a codec definition for a class 'MyData'
// which has a field 'BsonDocument extraData;'

// 1. Define the codec for known fields
BuilderCodec<MyData> baseCodec = BuilderCodec.builder(MyData::new)
    .field("id", Codecs.INTEGER, MyData::getId, MyData::setId)
    .field("name", Codecs.STRING, MyData::getName, MyData::setName)
    .build();

// 2. Wrap it with DocumentContainingCodec to handle unknown fields
Codec<MyData> finalCodec = new DocumentContainingCodec<>(
    baseCodec,
    MyData::setExtraData, // The setter for the catch-all field
    MyData::getExtraData  // The getter for the catch-all field
);
```

### Anti-Patterns (Do NOT do this)
- **Use in New Code:** The most critical anti-pattern is using this class at all. Its deprecated status indicates it has been superseded.
- **Direct Instantiation:** Do not instantiate this class directly in business logic. Codecs should be defined centrally and retrieved from a registry.
- **Overlapping Keys:** Ensure that the keys handled by the wrapped BuilderCodec do not overlap with keys that might appear in the "extra" data. If a key exists in both, the explicitly defined field from the BuilderCodec will take precedence during decoding, and the "extra" data for that key will be lost.

## Data Pipeline
This component acts as a filter and aggregator within the BSON serialization and deserialization data flow.

**Decoding (BSON to Java Object)**
> Flow:
> BSON Stream → BsonDocument → **DocumentContainingCodec** → (Known fields are removed) → Residual BsonDocument → Setter Function → Java Object Field

**Encoding (Java Object to BSON)**
> Flow:
> Java Object → Getter Function → Residual BsonDocument → **DocumentContainingCodec** → (Merged with BSON from known fields) → Final BsonDocument → BSON Stream

