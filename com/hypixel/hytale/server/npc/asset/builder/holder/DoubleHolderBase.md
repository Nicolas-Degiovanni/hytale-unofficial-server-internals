---
description: Architectural reference for DoubleHolderBase
---

# DoubleHolderBase

**Package:** com.hypixel.hytale.server.npc.asset.builder.holder
**Type:** Component / Abstract Base Class

## Definition
```java
// Signature
public abstract class DoubleHolderBase extends ValueHolder {
```

## Architecture & Concepts
DoubleHolderBase is an abstract component within the server-side NPC asset building framework. It serves as a specialized container for numeric values that are defined in a data-driven format, such as JSON. This class is not used directly but is extended by concrete implementations that represent specific NPC attributes like health, damage, or speed.

The core design principle is to support both static and dynamic values transparently. A value can be a simple, constant number or a complex mathematical expression that is resolved at runtime using an ExecutionContext. This allows for flexible and powerful NPC definitions where attributes can depend on game state (e.g., world level, player count).

A key architectural feature is its two-tiered validation system:
1.  **Intrinsic Validation:** A DoubleValidator is used to enforce constraints on the value itself, such as range checks (e.g., value must be between 0.0 and 100.0).
2.  **Relational Validation:** A list of ObjDoubleConsumer delegates can be attached to validate the value in relation to *other* system values. This is critical for ensuring logical consistency across multiple attributes (e.g., an NPC's minimum damage cannot exceed its maximum damage).

The class intelligently defers validation for dynamic expressions until runtime (`rawGet`), while validating static values immediately at parse time (`readJSON`). This is a deliberate performance optimization to fail-fast on invalid static configurations and avoid redundant checks on every access.

## Lifecycle & Ownership
-   **Creation:** Instances are not created directly. Concrete subclasses are instantiated by a higher-level asset parser or builder when it processes a numeric field from an NPC's JSON definition file.
-   **Scope:** The lifetime of a DoubleHolderBase instance is transient and strictly bound to a single asset-building session. It acts as an intermediate representation of a configured value before the final, immutable NPC asset is compiled.
-   **Destruction:** The object is eligible for garbage collection as soon as the asset building process completes and the final NPC data object is constructed. It holds no persistent references and is not intended to outlive the build phase.

## Internal State & Concurrency
-   **State:** The class is stateful and highly mutable during the configuration phase. It stores a DoubleValidator and a lazily-initialized list of relationValidators. The underlying value or expression is stored in the `expression` field inherited from the ValueHolder parent class.

-   **Thread Safety:** **This class is not thread-safe.** It is designed for use within a single-threaded asset-building pipeline. Concurrent calls to `addRelationValidator` or modifications from multiple threads will lead to race conditions and unpredictable behavior, especially due to the non-atomic lazy initialization of the `relationValidators` list.

    **WARNING:** All interactions with a DoubleHolderBase instance, from creation via `readJSON` to data retrieval via `rawGet`, must be performed by the same thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readJSON(json, validator, name, params) | void | O(1) | Configures the holder from a required JSON element. Validates static values immediately. |
| readJSON(json, default, validator, name, params) | void | O(1) | Configures the holder from an optional JSON element, using a default if absent. |
| addRelationValidator(validator) | void | O(1) | Registers a cross-field validator to be executed at runtime. |
| rawGet(executionContext) | double | O(E) | Resolves the expression and returns the raw double value. Triggers runtime validation for dynamic expressions. Complexity O(E) depends on the expression. |
| validate(value) | void | O(1) | Throws IllegalStateException if the value fails the intrinsic DoubleValidator check. |

## Integration Patterns

### Standard Usage
A concrete implementation of DoubleHolderBase is typically used as a field within a larger builder class. The builder orchestrates the parsing and linking of multiple holders.

```java
// In a hypothetical NpcAttributeBuilder class

// 1. Define holders for related attributes
private MaxHealthHolder maxHealth = new MaxHealthHolder();
private StartHealthHolder startHealth = new StartHealthHolder();

// 2. Parse values from JSON
JsonObject healthJson = npcJson.getAsJsonObject("health");
maxHealth.readJSON(healthJson.get("max"), maxHealthValidator, "maxHealth", params);
startHealth.readJSON(healthJson.get("start"), startHealthValidator, "startHealth", params);

// 3. Establish a relational validation rule
// Ensures starting health never exceeds maximum health
startHealth.addRelationValidator((context, startHealthValue) -> {
    double maxHealthValue = maxHealth.rawGet(context);
    if (startHealthValue > maxHealthValue) {
        throw new IllegalStateException("Start health cannot exceed max health.");
    }
});
```

### Anti-Patterns (Do NOT do this)
-   **Reusing Instances:** Do not attempt to reuse a holder instance across the construction of different NPC assets. Its internal state is tied to a single configuration pass.
-   **Concurrent Access:** Never access a holder from multiple threads. The asset-building pipeline must be single-threaded.
-   **Bypassing `rawGet`:** Do not attempt to access the internal `expression` field directly. The `rawGet` method contains critical, just-in-time validation logic for dynamic values that must be executed.

## Data Pipeline
The flow of data through this component follows a clear "parse-then-evaluate" model.

> Flow:
> JSON Configuration File → GSON Parser → `JsonElement` → `DoubleHolderBase.readJSON()` → **Populated DoubleHolderBase Instance** → Runtime `ExecutionContext` → `DoubleHolderBase.rawGet()` → Final `double` Value

