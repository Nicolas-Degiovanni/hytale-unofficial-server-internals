---
description: Architectural reference for TemporalArrayHolder
---

# TemporalArrayHolder

**Package:** com.hypixel.hytale.server.npc.asset.builder.holder
**Type:** Transient

## Definition
```java
// Signature
public class TemporalArrayHolder extends StringArrayHolder {
```

## Architecture & Concepts

The TemporalArrayHolder is a specialized data container and transformer within the server's NPC asset building framework. It is not a general-purpose utility but a stateful component designed to handle a specific data conversion task: transforming a JSON string array into a strongly-typed array of Java Time API objects.

Architecturally, this class serves as a validation and parsing layer on top of its parent, StringArrayHolder. While the parent class is responsible for handling the initial extraction of a string array from a JSON source (including the evaluation of dynamic expressions), TemporalArrayHolder is responsible for the subsequent steps:

1.  **Parsing:** It converts string representations of time (e.g., "10D", "PT20S") into concrete `Period` or `Duration` objects.
2.  **Validation:** It enforces constraints on the parsed temporal data, such as array length and custom validation logic provided by a TemporalArrayValidator.
3.  **Caching:** For performance, it implements a critical optimization. If the source data from the JSON is determined to be static (not containing dynamic expressions), the parsed `TemporalAmount` array is cached internally upon first read. Subsequent requests for the data will return the cached instance, avoiding costly re-parsing.

This component is fundamental to ensuring that time-based configurations for NPCs are both syntactically correct and semantically valid before they are used by the game logic.

### Lifecycle & Ownership

-   **Creation:** An instance of TemporalArrayHolder is created and configured exclusively by a higher-level asset builder during the deserialization of an NPC definition file. It is initialized via the `readJSON` method.
-   **Scope:** The object's lifetime is strictly bound to the asset building process. It exists as an intermediate container to hold and validate a single configuration property.
-   **Destruction:** It holds no external resources and is eligible for garbage collection as soon as the parent asset builder completes its work and is itself destroyed.

## Internal State & Concurrency

-   **State:** **Mutable and stateful**. The object is inert upon construction and is populated by the `readJSON` method. Its primary internal state is the `cachedTemporalArray`, which is set only if the underlying data is static.

-   **Thread Safety:** **Not thread-safe**. This class is designed for single-threaded use within the asset loading pipeline. Its internal fields, such as `validator` and `cachedTemporalArray`, are accessed and mutated without any synchronization mechanisms.

    **WARNING:** Sharing an instance of TemporalArrayHolder across multiple threads will result in race conditions and undefined behavior. It must be confined to the thread that is performing the asset loading.

## API Surface

The public API is minimal, exposing only the methods necessary for initialization and data retrieval within the builder framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readJSON(...) | void | O(N) | Initializes the holder from a JSON source, populating and validating its internal state. Caches the result if the source is static. |
| getTemporalArray(ctx) | TemporalAmount[] | O(1) or O(N) | Retrieves the final `TemporalAmount` array. Complexity is O(1) for static, cached data and O(N) for dynamic data that must be re-evaluated. |
| convertStringToTemporalArray(src) | static TemporalAmount[] | O(N) | The core static utility for converting a string array to a `TemporalAmount` array. Throws `IllegalStateException` on parsing failure. |

## Integration Patterns

### Standard Usage

This class is not intended for direct use by game logic developers. It is an internal component of the asset building system. A higher-level builder would use it as follows to process a JSON field.

```java
// Conceptual example within a hypothetical NPC asset builder

// 1. A JsonElement is extracted from the NPC definition file
JsonElement durationList = npcJson.get("behaviorCooldowns");
String propertyName = "behaviorCooldowns";

// 2. The builder creates and initializes the holder
TemporalArrayHolder holder = new TemporalArrayHolder();
holder.readJSON(
    durationList,
    1, // minLength
    10, // maxLength
    new MyCustomTemporalValidator(),
    propertyName,
    builderParameters
);

// 3. The holder is stored for later use
this.cooldownsHolder = holder;

// 4. Later, when the final NPC object is constructed...
ExecutionContext context = ...;
TemporalAmount[] cooldowns = this.cooldownsHolder.getTemporalArray(context);
npc.setBehaviorCooldowns(cooldowns);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new TemporalArrayHolder()` and then attempt to call `getTemporalArray`. The object is uninitialized and will fail. It must be configured via `readJSON`.
-   **Multi-threaded Access:** Do not pass an instance of this class between threads. The lack of synchronization makes it inherently unsafe for concurrent use.
-   **State Re-configuration:** Do not call `readJSON` more than once on the same instance. The behavior is undefined and may lead to inconsistent internal state.

## Data Pipeline

TemporalArrayHolder acts as a transformation stage in the NPC asset data pipeline. It consumes a potentially dynamic string array and produces a validated, strongly-typed temporal array.

> Flow:
> JSON File -> GSON Parser -> `JsonElement` -> `StringArrayHolder` (Expression Evaluation) -> **`TemporalArrayHolder`** (Parsing & Caching) -> `TemporalAmount[]` -> NPC Game Logic

