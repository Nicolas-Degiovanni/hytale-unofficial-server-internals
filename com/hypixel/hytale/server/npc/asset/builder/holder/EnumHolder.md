---
description: Architectural reference for EnumHolder
---

# EnumHolder

**Package:** com.hypixel.hytale.server.npc.asset.builder.holder
**Type:** Transient State Holder

## Definition
```java
// Signature
public class EnumHolder<E extends Enum<E>> extends StringHolderBase {
```

## Architecture & Concepts
The EnumHolder is a specialized component within the server's NPC asset building framework. Its primary function is to bridge the gap between raw string values defined in JSON asset files and the server's strongly-typed Java Enum constants. It is a fundamental building block for creating configurable and dynamic NPC behaviors.

Architecturally, EnumHolder serves as a smart container that encapsulates not just the final enum value, but also the logic required to resolve and validate it. It supports two distinct operational modes, determined by the underlying expression provided during initialization:

1.  **Static Resolution:** If the source JSON provides a simple string literal, EnumHolder parses and caches the corresponding enum constant once during the initial `readJSON` phase. All subsequent retrievals are near-zero-cost lookups from this cache. This is the most common and performant mode, used for static NPC properties.

2.  **Dynamic Resolution:** If the source JSON provides an expression, the EnumHolder defers resolution until a specific `ExecutionContext` is available (typically when an NPC is being spawned or updated). The expression is evaluated against the context, and the resulting string is converted to an enum. This powerful mechanism allows NPC properties to change based on game state, time of day, or other runtime factors.

A key feature is its extensible validation system. Other components in the asset builder can register `enumRelationValidators` to enforce complex, cross-field consistency rules. For example, a validator could ensure that if an NPC's `Faction` enum is set to `UNDEAD`, its `Diet` enum cannot be `HERBIVORE`.

## Lifecycle & Ownership
-   **Creation:** An EnumHolder is never created in isolation. It is instantiated and configured exclusively by a parent `BuilderBase` instance during the deserialization of an NPC asset from JSON. Its lifecycle is entirely managed by this parent builder.
-   **Scope:** The object's lifetime is tightly bound to the in-memory representation of the NPC asset definition. It persists as long as the asset template is held by the `AssetManager` or a similar system.
-   **Destruction:** The EnumHolder is eligible for garbage collection as soon as its parent builder is dereferenced. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** The EnumHolder is a stateful and mutable object. Its internal state includes:
    -   `enumConstants`: A cached array of all possible enum values for its generic type.
    -   `value`: A mutable cache for the resolved enum. For static holders, this is populated once at creation. For dynamic holders, this is updated on each call to `rawGet`.
    -   `enumRelationValidators`: A list of validation callbacks that can be added to post-creation.

-   **Thread Safety:** **This class is not thread-safe.** It is designed for single-threaded access during the asset loading and entity spawning phases of the server loop. The mutable `value` cache and the unsynchronized `enumRelationValidators` list make it susceptible to race conditions if accessed concurrently. All interactions with an EnumHolder instance must be synchronized externally or confined to a single thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readJSON(json, clazz, name, params) | void | O(1) | Configures the holder from a required JSON value. Caches the static value if possible. |
| readJSON(json, clazz, default, name, params) | void | O(1) | Configures the holder from an optional JSON value, using a default if absent. |
| get(ExecutionContext) | E | O(E + V) | Resolves and validates the enum value. This is the primary, safe access method. E is expression complexity; V is validator count. |
| addEnumRelationValidator(BiConsumer) | void | O(1) | Registers a new validation rule to be executed by the `get` method. |
| rawGet(ExecutionContext) | E | O(E) | Resolves the enum value but **skips all validation**. Intended for internal framework use only. |

## Integration Patterns

### Standard Usage
The EnumHolder is intended to be used by a parent builder class. The standard flow is to initialize it during JSON parsing and later resolve its value using a runtime context when an NPC is instantiated.

```java
// In a parent Builder class during asset parsing:
// Assume 'data' is a JsonObject from the asset file.
// 'params' is the current BuilderParameters.

// 1. Create and configure the holder for a required enum
EnumHolder<NpcState> stateHolder = new EnumHolder<>();
stateHolder.readJSON(data.get("state"), NpcState.class, "state", params);

// 2. Add a validator to enforce a rule
stateHolder.addEnumRelationValidator((context, state) -> {
    if (state == NpcState.AGGRESSIVE && !context.isHostileEnvironment()) {
        throw new IllegalStateException("Cannot be aggressive in a safe zone.");
    }
});

// 3. Later, when spawning an NPC instance:
ExecutionContext spawnContext = NpcSpawnEngine.createContext(...);
NpcState currentState = stateHolder.get(spawnContext); // Safely gets and validates the state
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation without Configuration:** `new EnumHolder()` creates a useless object. It must be configured via a `readJSON` call to be functional.
-   **Using rawGet for Business Logic:** Calling `rawGet` bypasses the critical validation checks registered via `addEnumRelationValidator`. This can lead to logically inconsistent NPC states and subtle bugs. Always prefer `get` unless you are part of the low-level builder framework that guarantees later validation.
-   **Mismatched Context:** For a dynamic holder (one based on an expression), passing a null or incomplete `ExecutionContext` to `get` will result in a `NullPointerException` or an incorrectly resolved value.

## Data Pipeline
The flow of data from a raw asset file to a final, validated enum value is a multi-stage process orchestrated by the EnumHolder.

> Flow:
> JSON String -> `BuilderBase` Deserialization -> **EnumHolder.readJSON()** -> Internal State (Expression + Enum Constants) -> **EnumHolder.get(context)** -> Expression Evaluation -> String-to-Enum Conversion -> **Validation Checks** -> Final, Typed Enum Value

---

