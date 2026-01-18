---
description: Architectural reference for EnumArrayHolder
---

# EnumArrayHolder

**Package:** com.hypixel.hytale.server.npc.asset.builder.holder
**Type:** Transient Data Holder

## Definition
```java
// Signature
public class EnumArrayHolder<E extends Enum<E>> extends ArrayHolder {
```

## Architecture & Concepts
The EnumArrayHolder is a specialized component within the server's NPC Asset Builder framework. Its primary function is to act as a strongly-typed container for an array of enum values that are parsed from a raw JSON configuration file.

This class serves as a critical bridge between the loosely-typed string representations in asset files and the type-safe enum arrays required by the server's runtime logic. It encapsulates the logic for parsing, converting, and validating these values.

By extending ArrayHolder, it inherits support for both static values (defined directly in the JSON) and dynamic values (resolved at runtime via an expression and an ExecutionContext). This allows for flexible and powerful NPC definitions where properties can change based on in-game conditions.

## Lifecycle & Ownership
-   **Creation:** An EnumArrayHolder is instantiated directly by a parent builder class, such as an NpcDefinitionBuilder, during the parsing phase of an asset file. It is never created as a standalone service or singleton.
-   **Scope:** The lifecycle of an EnumArrayHolder is strictly bound to its parent builder. It exists only for the duration of a single asset-building operation.
-   **Destruction:** The object is marked for garbage collection as soon as the parent builder completes its work and the final, constructed asset (e.g., an NpcDefinition) is returned. It holds no persistent state beyond this process.

## Internal State & Concurrency
-   **State:** This class is highly stateful and mutable during its initialization phase. It caches the enum class type, its constants, and an optional validator. The core state, the resolved enum array named *value*, is populated lazily on the first call to *get*. Once resolved, this *value* is cached and is not re-calculated, effectively becoming immutable for the remainder of the object's short lifecycle.
-   **Thread Safety:** **This class is not thread-safe.** It is designed for synchronous, single-threaded use within the asset loading pipeline. Concurrent access, especially simultaneous calls to *get* or *resolve* on an uninitialized instance, will result in race conditions and undefined behavior. All operations on an instance must be performed by the same thread.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readJSON(json, class, validator, name, params) | void | O(1) | Configures the holder with its source data and validation rules. Must be called before any other method. |
| get(executionContext) | E[] | O(N) first call, O(1) after | Resolves the underlying string array into a typed enum array and returns it. Caches the result. |
| validate(executionContext) | void | O(N) | A lifecycle method to force resolution and validation of the held value. |
| resolve(value) | void | O(N) | Internal method that performs the string-to-enum conversion and validation. Throws IllegalStateException on validation failure. |

## Integration Patterns

### Standard Usage
The EnumArrayHolder is intended to be created and used entirely within the scope of a larger builder class that reads from a JSON source.

```java
// Inside a parent builder class responsible for parsing a JSON object

// 1. Instantiate the holder
EnumArrayHolder<NpcBehavior> behaviors = new EnumArrayHolder<>();

// 2. Configure it from a JSON source
behaviors.readJSON(
    jsonObject.get("behaviors"),
    NpcBehavior.class,
    new EnumArrayValidator.Unique(), // Optional: enforce unique values
    "behaviors",
    builderParameters
);

// 3. During the final object construction, get the resolved value
//    The executionContext is used for dynamic, expression-based values.
NpcBehavior[] resolvedBehaviors = behaviors.get(executionContext);
finalNpc.setBehaviors(resolvedBehaviors);
```

### Anti-Patterns (Do NOT do this)
-   **Reusing Instances:** Do not attempt to reuse an EnumArrayHolder instance for multiple JSON properties. Each instance is stateful and designed to manage a single configuration value. Create a new instance for each property.
-   **Access Before Configuration:** Calling *get* or *validate* before *readJSON* has been successfully invoked will result in a NullPointerException or other unhandled exceptions.
-   **External Instantiation:** This class should not be instantiated outside of the asset building system. It is not a general-purpose data structure.

## Data Pipeline
The EnumArrayHolder is a key transformation step in the data pipeline that converts raw text configuration into executable server logic.

> Flow:
> JSON Asset File -> GSON Parser -> JsonElement -> **EnumArrayHolder.readJSON** -> **EnumArrayHolder.resolve** -> `E[]` (Typed Enum Array) -> Final NPC Asset Object

