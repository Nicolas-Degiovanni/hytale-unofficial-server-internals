---
description: Architectural reference for NumberArrayHolder
---

# NumberArrayHolder

**Package:** com.hypixel.hytale.server.npc.asset.builder.holder
**Type:** Transient

## Definition
```java
// Signature
public class NumberArrayHolder extends ArrayHolder {
```

## Architecture & Concepts
The NumberArrayHolder is a specialized, stateful container within the server's NPC Asset Builder framework. Its primary function is to represent, parse, and validate a numeric array property (either integer or double) from an NPC's raw JSON definition.

This class acts as a critical bridge between the untyped, schema-less world of JSON configuration and the strongly-typed, validated data required by the server's runtime NPC systems. It encapsulates not only the potential value of a property but also its associated validation rules, such as array length and element constraints.

A key architectural concept inherited from its parent, ArrayHolder, is the distinction between *static* and *dynamic* values.
*   **Static values** are constant literals defined directly in the JSON, which are parsed, validated once at load time, and then cached.
*   **Dynamic values** are defined as expressions that are resolved at runtime using an ExecutionContext. This allows NPC properties to change based on game state, time of day, or other runtime factors. Validation for dynamic values is deferred until each time the value is requested.

## Lifecycle & Ownership
- **Creation:** A new NumberArrayHolder is instantiated by a higher-level asset builder whenever a numeric array property is encountered during the deserialization of an NPC asset file. An instance is created for each unique property.
- **Scope:** The object's lifecycle is strictly bound to the parent asset builder. It exists only for the duration of the asset loading and validation process. It is a short-lived, transient object, not a persistent component of the game state.
- **Destruction:** The instance is marked for garbage collection as soon as the NPC asset is fully constructed and its configuration has been transferred to a final NPC template or entity. It holds no unmanaged resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The NumberArrayHolder is highly mutable during its initialization phase via the readJSON methods. It stores the parsed expression, a reference to a validator object (either IntArrayValidator or DoubleArrayValidator), and metadata like the property name. After initialization, the underlying value it represents may be immutable (for static expressions) or recalculated on each access (for dynamic expressions).

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed exclusively for use within the single-threaded context of the asset loading pipeline. Concurrent calls to its methods, particularly readJSON and get, will result in race conditions and undefined behavior. All interactions must be externally synchronized or confined to the main asset-loading thread.

## API Surface
The public API is centered around data ingestion (readJSON) and retrieval (get, getIntArray).

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readJSON(...) | void | O(N) | Overloaded methods to parse a JsonElement. Populates the internal expression and assigns validators. Throws if JSON is malformed. N is the number of elements in the array. |
| get(context) | double[] | O(E) | Resolves the expression and returns the value as a double array. Triggers runtime validation for dynamic expressions. E is the complexity of the underlying expression. |
| getIntArray(context) | int[] | O(E) | Resolves the expression and returns the value as an integer array. Triggers runtime validation for dynamic expressions. E is the complexity of the underlying expression. |
| validate(context) | void | O(N) | Explicitly triggers the validation logic. This is typically called internally by get methods. |

## Integration Patterns

### Standard Usage
The NumberArrayHolder is not intended for direct use by game logic developers. It is an internal component of the asset building system. A builder class would use it to process a specific JSON field.

```java
// Example from within a hypothetical NpcAssetBuilder
NumberArrayHolder spawnParticleVelocity = new NumberArrayHolder();
JsonElement velocityData = npcJsonDefinition.get("spawnParticleVelocity");
BuilderParameters params = this.getBuilderParameters();

// Ingest and configure the holder from JSON
spawnParticleVelocity.readJSON(
    velocityData,
    3, 3, // min/max length
    new DoubleRangeValidator(-10.0, 10.0), // validator
    "spawnParticleVelocity", // property name for error messages
    params
);

// Later, when an NPC instance is being created...
ExecutionContext context = NpcRuntime.createContext(npc);
double[] velocity = spawnParticleVelocity.get(context);
world.spawnParticle(..., velocity);
```

### Anti-Patterns (Do NOT do this)
- **Post-Initialization Modification:** Do not attempt to modify a NumberArrayHolder after the initial readJSON call. The object is not designed for reuse or reconfiguration. Create a new instance for each property.
- **Ignoring ExecutionContext:** Passing a null or stale ExecutionContext to get or getIntArray for a dynamic expression will result in incorrect values or runtime exceptions.
- **Premature Access:** Calling get or getIntArray before readJSON has been successfully executed will result in a NullPointerException, as the internal expression object will not have been created.

## Data Pipeline
The NumberArrayHolder is a specific stage in the data transformation pipeline that converts raw text configuration into live game objects.

> Flow:
> NPC JSON File -> GSON Parser -> JsonElement -> **NumberArrayHolder.readJSON** -> Internal Expression & Validators -> **NumberArrayHolder.get** -> Validated double[] or int[] -> NPC Runtime Component<ctrl63>

