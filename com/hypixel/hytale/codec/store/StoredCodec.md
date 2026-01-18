---
description: Architectural reference for StoredCodec
---

# StoredCodec

**Package:** com.hypixel.hytale.codec.store
**Type:** Transient Proxy

## Definition
```java
// Signature
public class StoredCodec<T> implements Codec<T> {
```

## Architecture & Concepts
The StoredCodec is a fundamental component in the Hytale serialization framework, acting as a **proxy** or **indirection layer**. It does not perform any encoding or decoding itself. Instead, it holds a reference, a CodecKey, to another codec that is registered in a central CodecStore.

This design pattern enables **deferred resolution** or **late binding** of codecs. It decouples a component that *needs* a codec from the component that *provides* it. This is critical for serializing complex object graphs, especially those with circular or forward dependencies, where defining all codecs in a linear, dependency-first order is impossible.

A system can be configured with a StoredCodec instance long before the target codec it points to is actually available in the CodecStore. The resolution from key to concrete codec implementation happens just-in-time when a serialization or deserialization operation is invoked.

### Lifecycle & Ownership
- **Creation:** A StoredCodec is a lightweight value object instantiated directly via its constructor: `new StoredCodec(someKey)`. It is typically created during the construction of a more complex, composite codec that needs to reference another codec by its key without creating a hard dependency.
- **Scope:** The lifetime of a StoredCodec instance is tied to the object that holds it. It contains no managed resources and can be freely created and garbage collected.
- **Destruction:** Managed entirely by the Java garbage collector. No explicit cleanup is required.

## Internal State & Concurrency
- **State:** The internal state is **immutable**. It consists of a single `final` field, the CodecKey, which is set at construction time and never changes.
- **Thread Safety:** This class is inherently thread-safe. All operations are read-only with respect to the instance's state. The overall thread safety of an operation depends on the thread safety of two external components:
    1. The provided CodecStore instance.
    2. The underlying Codec that is resolved from the store.

    **Warning:** Concurrent modification of the CodecStore while a StoredCodec is actively resolving a key can lead to unpredictable behavior. The CodecStore is expected to be fully populated before serialization operations begin.

## API Surface
The API surface mirrors the Codec interface. All methods delegate to the real codec instance after resolving it from the CodecStore.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(bson, info) | T | O(L+C) | Resolves the real codec and delegates the BSON decode operation. Throws IllegalArgumentException if the key is not found in the store. L is the lookup time, C is the complexity of the real codec. |
| encode(t, info) | BsonValue | O(L+C) | Resolves the real codec and delegates the BSON encode operation. Throws IllegalArgumentException if the key is not found in the store. |
| decodeJson(reader, info) | T | O(L+C) | Resolves the real codec and delegates the JSON decode operation. Throws IllegalArgumentException if the key is not found in the store. |
| toSchema(context) | Schema | O(L+C) | Resolves the real codec from the **static** CodecStore and delegates schema generation. This is a critical distinction from other methods. |

## Integration Patterns

### Standard Usage
The primary use case is to break circular dependencies or defer codec resolution within a larger codec definition.

```java
// Example: A PlayerCodec needs an InventoryCodec, but InventoryCodec might need a PlayerCodec.
// Using StoredCodec breaks this dependency cycle.

// In some central registration location:
CodecKey<Inventory> inventoryKey = new CodecKey<>("core:inventory");
codecStore.register(inventoryKey, new InventoryCodec());

// In the PlayerCodec definition:
public class PlayerCodec implements Codec<Player> {
    private final Codec<Inventory> inventoryCodec = new StoredCodec<>(new CodecKey<>("core:inventory"));

    @Override
    public Player decode(BsonValue bson, ExtraInfo extraInfo) {
        // ... decode player fields
        BsonValue inventoryBson = bson.asDocument().get("inventory");
        Inventory inv = this.inventoryCodec.decode(inventoryBson, extraInfo); // Resolves at runtime
        // ...
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Unregistered Keys:** Do not use a StoredCodec with a CodecKey that has not been registered in the target CodecStore. This will result in a runtime IllegalArgumentException. All keys must be registered before use.
- **Redundant Indirection:** Avoid using StoredCodec when a direct reference to the concrete codec is already available and does not create dependency issues. This adds unnecessary lookup overhead.
- **Static Store Mismatch:** The `toSchema` method uniquely uses `CodecStore.STATIC`. If this static store is not populated identically to the runtime store passed in ExtraInfo, schema generation will not accurately reflect runtime behavior. This can lead to severe data validation failures.

## Data Pipeline
StoredCodec acts as a routing or dispatch node in the data pipeline. It does not transform data but instead directs it to the appropriate component for processing.

> **Decoding Flow:**
> BsonValue -> `StoredCodec.decode` -> CodecStore lookup -> **RealCodec**.decode -> Java Object

> **Encoding Flow:**
> Java Object -> `StoredCodec.encode` -> CodecStore lookup -> **RealCodec**.encode -> BsonValue

