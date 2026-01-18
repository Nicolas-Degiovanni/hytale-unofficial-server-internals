---
description: Architectural reference for BsonFunctionCodec
---

# BsonFunctionCodec

**Package:** com.hypixel.hytale.codec.function
**Type:** Utility / Decorator

## Definition
```java
// Signature
@Deprecated
public class BsonFunctionCodec<T> implements Codec<T>, WrappedCodec<T> {
```

## Architecture & Concepts

The BsonFunctionCodec is a legacy implementation of the Decorator pattern for the Hytale serialization framework. Its primary role is to augment the behavior of an existing Codec by applying custom transformation logic during the serialization and deserialization processes.

It acts as a wrapper around a child Codec, intercepting the data flow to and from that codec. This allows for on-the-fly data manipulation without altering the core logic of the wrapped component. For example, it can be used to inject default values into a decoded object or to transform a data structure into a legacy format before encoding.

**WARNING:** This class is marked as **Deprecated**. It represents an older pattern for Codec modification and should not be used in new development. Modern systems should favor more explicit and type-safe composition patterns provided by the core Codec library.

From a data contract perspective, this codec is transparent. It delegates all schema generation requests directly to the wrapped child codec, meaning it does not and cannot alter the formal schema of the data it processes.

## Lifecycle & Ownership

-   **Creation:** Instantiated directly in code, typically during the construction of a Codec registry or a component that requires specialized serialization behavior. It is not managed by a service locator or dependency injection framework.
-   **Scope:** The lifecycle of a BsonFunctionCodec instance is ephemeral and tied directly to the component that created it. It is a lightweight object that holds no resources.
-   **Destruction:** The object is eligible for garbage collection as soon as all references to it are released. There is no explicit cleanup or disposal method.

## Internal State & Concurrency

-   **State:** **Immutable**. The internal fields, which consist of the wrapped codec and two BiFunction instances, are declared final and are assigned only once during construction. This class does not cache data or maintain any mutable state between calls.

-   **Thread Safety:** **Conditionally Thread-Safe**. The BsonFunctionCodec itself introduces no concurrency hazards. However, its overall thread safety is entirely dependent on the guarantees provided by the wrapped Codec and the supplied encode and decode functions. If the wrapped components are thread-safe, this class can be safely shared across multiple threads.

    **WARNING:** Functions provided to the constructor that modify shared state will break thread-safety guarantees and lead to unpredictable behavior.

## API Surface

The public API is defined by the Codec and WrappedCodec interfaces.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(bsonValue, extraInfo) | T | O(N+F) | Decodes a BsonValue using the child codec, then applies the custom decode function to the result. |
| encode(r, extraInfo) | BsonValue | O(N+F) | Encodes an object using the child codec, then applies the custom encode function to the resulting BsonValue. |
| toSchema(context) | Schema | O(N) | Delegates schema generation directly to the wrapped child codec. |
| getChildCodec() | Codec<T> | O(1) | Returns the wrapped child codec instance. |

*Complexity Key: N = Complexity of the wrapped codec, F = Complexity of the transformation function.*

## Integration Patterns

### Standard Usage

The intended use is to wrap an existing codec to inject simple, stateless transformations.

```java
// Assume 'playerStateCodec' is an existing Codec<PlayerState>
// This transformation ensures a 'lastSeen' timestamp is added upon decoding.

Codec<PlayerState> legacyCodec = new BsonFunctionCodec<>(
    playerStateCodec,
    (playerState, bsonValue) -> {
        // Post-decode logic: apply a default or calculated value.
        playerState.setLastSeen(System.currentTimeMillis());
        return playerState;
    },
    (bsonValue, playerState) -> {
        // Pre-encode logic: no-op in this case, return as-is.
        return bsonValue;
    }
);

// This 'legacyCodec' can now be used in place of the original.
```

### Anti-Patterns (Do NOT do this)

-   **Usage in New Code:** The most significant anti-pattern is using this class at all. Its deprecated status indicates that a superior, officially supported alternative exists.
-   **Stateful Transformations:** Do not provide BiFunctions that rely on or modify external state. This breaks the immutability and thread-safety assumptions of the codec system and can lead to severe concurrency bugs.
-   **Schema Violation:** The transformation functions must not produce a BsonValue that violates the schema of the wrapped codec. Doing so will cause downstream validation errors or data corruption.
-   **Complex Business Logic:** Avoid embedding complex, service-dependent logic within the transformation functions. These functions should be lightweight, pure, and focused solely on data shape transformation.

## Data Pipeline

BsonFunctionCodec acts as an intermediate processing step in the serialization and deserialization data flows.

**Decoding Pipeline:**
> Flow:
> Raw BsonValue -> Wrapped Codec `decode` -> Intermediate Object `T` -> **BsonFunctionCodec `decode` function** -> Final Object `T`

**Encoding Pipeline:**
> Flow:
> Object `T` -> Wrapped Codec `encode` -> Intermediate BsonValue -> **BsonFunctionCodec `encode` function** -> Final BsonValue

