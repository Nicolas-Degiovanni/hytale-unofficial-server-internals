---
description: Architectural reference for StringArrayHolder
---

# StringArrayHolder

**Package:** com.hypixel.hytale.server.npc.asset.builder.holder
**Type:** Transient

## Definition
```java
// Signature
public class StringArrayHolder extends ArrayHolder {
```

## Architecture & Concepts
The StringArrayHolder is a specialized state container within the server-side NPC asset building framework. Its primary function is to manage the lifecycle of a single string array property defined within an NPC's JSON configuration file.

This class is not a simple data wrapper. It serves as a sophisticated validation and resolution engine for a property that can be either a static, literal array or a dynamic value computed from an expression. It decouples the initial parsing of a value from its final, context-aware validation.

The core architectural pattern is a **two-phase validation system**:

1.  **Phase 1: Structural Validation:** Performed immediately during the `readJSON` call. This phase validates the fundamental structure of the data: array length constraints and per-element format rules defined by a StringArrayValidator. This fails fast if the JSON provides a malformed value.
2.  **Phase 2: Relational Validation:** Performed on-demand when the `get` method is invoked. This phase executes a series of registered validators that can inspect the array's contents in the context of other, already-parsed NPC properties. This allows for complex, cross-field dependency checks, such as ensuring that a list of equipment tags corresponds to actual items defined elsewhere.

This two-phase approach is critical for handling complex asset files where the validity of one property depends on the value of another.

## Lifecycle & Ownership
-   **Creation:** A StringArrayHolder is instantiated by a parent builder class (e.g., an `NPCAssetBuilder`) during the traversal of an NPC's JSON asset tree. It is created specifically to manage one property.
-   **Scope:** The object's lifetime is strictly bound to the parsing and construction of a single NPC asset. It holds intermediate state required for validation but does not persist after the final NPC object is built.
-   **Destruction:** The instance becomes eligible for garbage collection as soon as the parent asset builder completes its operation and releases its reference. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** The StringArrayHolder is a mutable, stateful component. Its internal state, including length constraints, validators, and the underlying expression, is configured post-construction via the `readJSON` and `addRelationValidator` methods. It does not cache the resolved value from the expression; each call to `get` or `rawGet` triggers a re-evaluation.

-   **Thread Safety:** **This class is not thread-safe.** It is designed for synchronous, single-threaded use within the asset building pipeline. The internal collections, such as `relationValidators`, are not protected by locks. Concurrent modification or access will lead to race conditions and undefined behavior.

    **WARNING:** Do not share instances of StringArrayHolder or the parent asset builder across multiple threads.

## API Surface
The public API is designed around a clear configure-then-read pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readJSON(...) | void | O(1) | Configures the holder from a JsonElement. Establishes length constraints and the primary validator. Throws if static validation fails. |
| get(ExecutionContext) | String[] | O(N) | Resolves the expression, runs all structural and relational validators, and returns the final value. N is the number of relation validators. This is the primary method for safe data retrieval. |
| rawGet(ExecutionContext) | String[] | O(1) | Resolves the expression and runs only the initial structural validation. **Bypasses relational checks.** Use with extreme caution. |
| addRelationValidator(...) | void | O(1) | Registers a new relational validator. This is the primary extension point for injecting cross-field validation logic. |

## Integration Patterns

### Standard Usage
The StringArrayHolder is intended to be used exclusively by a higher-level builder class. The builder orchestrates the parsing, registration of cross-field validators, and final retrieval of the value.

```java
// Within a hypothetical NPCAssetBuilder...

// 1. Create the holder for a specific property
StringArrayHolder equipmentTags = new StringArrayHolder();

// 2. Configure it from the parsed JSON
JsonElement tagsElement = npcJson.get("equipmentTags");
equipmentTags.readJSON(tagsElement, 1, 5, new ItemTagValidator(), "equipmentTags", params);

// 3. Register a validator that depends on another property (e.g., faction)
StringHolder faction = ... // another holder for the faction property
equipmentTags.addRelationValidator((context, tags) -> {
    if ("undead".equals(faction.get(context)) && !hasTag(tags, "cursed")) {
        throw new IllegalStateException("Undead faction requires 'cursed' equipment tag.");
    }
});

// 4. Later, during the final build step, retrieve the fully validated value
String[] finalTags = equipmentTags.get(buildContext);
npc.setEquipmentTags(finalTags);
```

### Anti-Patterns (Do NOT do this)
-   **Bypassing Relational Validation:** Calling `rawGet` to retrieve the value is dangerous. It skips the critical cross-field validation step, potentially leading to an NPC with an inconsistent or invalid state that will cause runtime errors. Always prefer `get`.
-   **Post-Read Configuration:** Do not call `addRelationValidator` after `get` has already been called for a given build context. The validation pipeline is not designed for re-entrancy.
-   **External Instantiation:** While technically possible, creating a StringArrayHolder outside of a parent builder workflow serves no purpose and breaks the intended ownership model.
-   **Concurrent Access:** Never access a single StringArrayHolder instance from multiple threads. The asset building process must be single-threaded.

## Data Pipeline
The StringArrayHolder acts as a validation and resolution stage within the larger NPC asset data pipeline.

> Flow:
> NPC JSON File -> GSON Parser -> Builder -> **StringArrayHolder.readJSON** -> **StringArrayHolder.addRelationValidator** -> Build Step -> **StringArrayHolder.get** -> Final NPC Object Property

