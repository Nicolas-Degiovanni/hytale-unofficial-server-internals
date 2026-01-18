---
description: Architectural reference for BuilderCombatConfig
---

# BuilderCombatConfig

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient Helper

## Definition
```java
// Signature
public class BuilderCombatConfig extends BuilderCodecObjectHelper<String> {
```

## Architecture & Concepts
The BuilderCombatConfig is a specialized, transient component within the server-side NPC asset loading pipeline. Its primary function is to resolve the name of an NPC's combat configuration, which dictates its fighting style, abilities, and balancing parameters.

Architecturally, this class embodies the principle of **Context-Sensitive Configuration**. It does not simply read a static value from a data file. Instead, it acts as a resolution point that intelligently merges a base configuration from an asset with potential runtime overrides. This allows the system to create NPC variants (e.g., a boss in a "hard mode" dungeon) that use different combat logic without requiring entirely separate NPC asset files.

This class is not intended for general-purpose use; it is an internal helper tightly coupled to the NPC deserialization and validation process. Its existence is an implementation detail of the asset system, designed to be orchestrated by a higher-level asset factory or parser.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the core NPC asset parsing framework when it encounters a `combatConfig` field within an NPC's JSON definition. It is never created directly by game logic systems.
-   **Scope:** The lifecycle of a BuilderCombatConfig instance is extremely brief. It exists only for the duration of parsing and validating a single NPC asset.
-   **Destruction:** Once the parent NPC object is fully constructed and validated, the BuilderCombatConfig instance is no longer referenced and becomes eligible for garbage collection. It does not persist in memory.

## Internal State & Concurrency
-   **State:** This object is stateful. It maintains an internal `value` (the config name from the asset file) and an `inline` flag after the `readConfig` method is called. This state is critical for its operation but is only valid within the narrow scope of a single asset load operation.

-   **Thread Safety:** **This class is not thread-safe.** It is designed to be created, used, and discarded within the context of a single thread processing an asset file. Its mutable internal state and the required sequence of method calls (`readConfig` followed by `build` or `validate`) make concurrent access from multiple threads an unsupported and dangerous operation that will lead to unpredictable behavior.

## API Surface
The public contract is designed for a specific, sequential workflow orchestrated by the asset loader.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(data, extraInfo) | void | O(1) | Deserializes a JSON fragment to populate the builder's internal state. Must be called before any other method. |
| build(context) | String | O(1) | Resolves the final combat config name. It critically prioritizes an override from the ExecutionContext over the value from the asset file. Returns null if no config is defined. |
| validate(name, helper, context, errors) | boolean | O(1) | Validates that the resolved combat config name corresponds to an existing BalanceAsset. Reports errors to the provided list. |
| build() | String | N/A | **Unsupported.** Throws UnsupportedOperationException. This method is intentionally disabled to enforce context-aware building. |

## Integration Patterns

### Standard Usage
This class is exclusively used by the internal asset loading system. The standard pattern involves deserialization, context-aware building, and validation.

```java
// Executed within a higher-level NPC asset factory...
ExecutionContext npcContext = createNpcContextForVariant(variantId);
List<String> validationErrors = new ArrayList<>();

// The factory instantiates the builder for a specific JSON field.
BuilderCombatConfig combatBuilder = new BuilderCombatConfig(codec, validator);

// 1. The JSON data is read into the builder.
combatBuilder.readConfig(npcJson.get("combatConfig"), assetExtraInfo);

// 2. The final config name is resolved using the runtime context.
String finalCombatConfig = combatBuilder.build(npcContext);

// 3. The resolved name is validated against global registries.
boolean isValid = combatBuilder.validate("MyCoolNPC", validationHelper, npcContext, validationErrors);
if (!isValid) {
    // Handle asset loading failure...
}
```

### Anti-Patterns (Do NOT do this)
-   **Calling Parameterless build:** Never call the `build()` method without an ExecutionContext. It is explicitly unsupported and will cause a server crash. This is a design choice to prevent developers from forgetting the critical override mechanism.
-   **Reusing Instances:** Do not attempt to reuse a BuilderCombatConfig instance to process multiple configurations. Its internal state is not designed to be reset, and doing so will result in corrupted or incorrect NPC data.
-   **Direct Instantiation:** Game logic developers must not instantiate this class directly using `new`. It is an internal component of the asset system, and its dependencies (Codec, Validator) are managed by that system.

## Data Pipeline
The BuilderCombatConfig acts as a transformation and validation stage in the larger NPC data pipeline. It takes raw configuration data and resolves it into a verified, context-aware reference.

> Flow:
> NPC Asset JSON -> Asset Deserializer -> **BuilderCombatConfig**.readConfig() -> **BuilderCombatConfig**.build(ExecutionContext) -> Resolved Combat Config Name (String) -> NPC Object Construction

