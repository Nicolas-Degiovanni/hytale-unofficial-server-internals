---
description: Architectural reference for BuilderCodecObjectHelper
---

# BuilderCodecObjectHelper

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient Helper

## Definition
```java
// Signature
public class BuilderCodecObjectHelper<T> {
```

## Architecture & Concepts
The BuilderCodecObjectHelper is a generic, stateful utility class that serves a critical role in the server-side asset and configuration loading pipeline. Its primary function is to orchestrate the deserialization and validation of a single JSON configuration block into a strongly-typed Java object of type T.

It acts as a single-use container that encapsulates the logic for:
1.  Translating a GSON JsonElement into a BSON representation.
2.  Decoding the BSON data into a Java object using a provided Codec.
3.  Validating the resulting object against a set of rules using a provided Validator.

By abstracting this common three-step process, it provides a standardized, reusable pattern for parsing complex configuration files, particularly for NPC assets as indicated by its package. It is a fundamental building block for higher-level asset builders that need to process structured data from disk or network sources.

### Lifecycle & Ownership
- **Creation:** An instance is created directly via its constructor (`new BuilderCodecObjectHelper(...)`). It is typically instantiated by a parent asset factory or builder that is iterating through a larger JSON configuration structure. It is not managed by a dependency injection container or service registry.
- **Scope:** The object's lifetime is intentionally brief. It is designed to process exactly one JsonElement. Its scope is confined to the method in which it is created, used to parse a configuration section, and then immediately discarded.
- **Destruction:** It becomes eligible for garbage collection as soon as the reference to it goes out of scope, which is typically after the `build` method has been called and the resulting object has been retrieved.

## Internal State & Concurrency
- **State:** This class is mutable and highly stateful. Its core state is the `value` field, which transitions from `null` to an instance of `T` upon a successful call to `readConfig`. This is a one-way state transition; the object is not designed to be reset or reused. The `codec` and `validator` fields are immutable after construction.

- **Thread Safety:** **This class is not thread-safe.** The internal `value` field is written to and read from by public methods without any synchronization. Concurrent calls to `readConfig` on the same instance will result in a race condition, where the final state of `value` is unpredictable. Instances must not be shared across threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| BuilderCodecObjectHelper(classType, codec, validator) | constructor | O(1) | Creates a new helper instance with the required deserialization and validation components. |
| build() | T | O(1) | Returns the deserialized and validated object. Returns null if `readConfig` has not been called. |
| readConfig(data, extraInfo) | void | O(N) | The primary operational method. Deserializes the JsonElement, validates the result, and stores it internally. Throws exceptions if validation fails catastrophically. |
| hasValue() | boolean | O(1) | A simple check to determine if `readConfig` has successfully populated the internal value. |

## Integration Patterns

### Standard Usage
The intended pattern is to instantiate, read, build, and discard. This ensures a clean, predictable state for each configuration block being processed.

```java
// Assume 'npcComponentJson' is a JsonElement from a config file
// and 'context' provides necessary ExtraInfo.

Codec<NpcBehavior> behaviorCodec = ...;
Validator<NpcBehavior> behaviorValidator = ...;

// 1. Instantiate the helper for a specific type
BuilderCodecObjectHelper<NpcBehavior> helper = new BuilderCodecObjectHelper<>(
    NpcBehavior.class,
    behaviorCodec,
    behaviorValidator
);

// 2. Process the JSON data
helper.readConfig(npcComponentJson, context.getExtraInfo());

// 3. Retrieve the final, validated object
if (helper.hasValue()) {
    NpcBehavior behavior = helper.build();
    // ... use the behavior object
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Reuse:** Do not call `readConfig` multiple times on the same instance. This will overwrite the internal `value` and can lead to subtle bugs if the old value was expected to be preserved elsewhere. Create a new helper for each distinct JSON object.
- **Premature Build:** Do not call `build` before `readConfig`. It will always return `null`, which may lead to a NullPointerException in downstream code that does not properly check the result.
- **Concurrent Access:** Do not share an instance of this helper across multiple threads. The lack of synchronization will lead to race conditions and unpredictable behavior.

## Data Pipeline
The helper manages a straightforward, linear flow of data from raw JSON to a validated Java object.

> Flow:
> JsonElement -> BsonUtil Translation -> BSON -> **Codec<T>.decode()** -> Raw Object `T` -> **Validator<T>.accept()** -> Validated Object `T`

