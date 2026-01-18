---
description: Architectural reference for ReputationRequirement
---

# ReputationRequirement

**Package:** com.hypixel.hytale.builtin.adventure.reputation.choices
**Type:** Transient Data Model

## Definition
```java
// Signature
public class ReputationRequirement extends ChoiceRequirement {
```

## Architecture & Concepts
The ReputationRequirement class is a concrete implementation of the ChoiceRequirement contract. Its primary function is to act as a data-driven gate for in-game choices, such as dialogue options or quest branches. It allows game designers to restrict content until a player has achieved a specific standing with a faction or group.

This class is not intended for direct, programmatic use. Instead, it is designed to be deserialized from game asset files (e.g., JSON) at runtime. The static CODEC field is the cornerstone of this design, providing a robust mechanism for parsing, instantiating, and validating the requirement against the master asset database.

The CODEC's integrated validators for ReputationGroup and ReputationRank are critical. They ensure data integrity at load time, guaranteeing that any configured requirement refers to valid, existing reputation groups and ranks. This preemptively catches configuration errors and prevents runtime failures. In essence, this class bridges the gap between static game data and the dynamic player state managed by the ReputationPlugin.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale Codec system during the asset loading phase. The framework reads a definition from a configuration file and uses the static CODEC to construct a corresponding in-memory object. Manual instantiation is a design violation.
- **Scope:** The object's lifetime is bound to the asset that contains it, such as a dialogue tree or quest definition. It is effectively an immutable data container that persists as long as its parent object is held in memory.
- **Destruction:** The object is managed by the Java garbage collector. It is reclaimed when its parent asset is unloaded and no longer referenced. No manual cleanup is necessary.

## Internal State & Concurrency
- **State:** Immutable. The internal fields, reputationGroupId and minRequiredRankId, are populated once during deserialization and are not intended to be modified thereafter. The class itself holds no mutable state.
- **Thread Safety:** The class is inherently thread-safe due to its immutable state. The canFulfillRequirement method is a pure function with respect to its own state. However, it reads from external systems like the EntityStore via the ReputationPlugin. The caller is responsible for ensuring that any read operations on the player's state are performed within the appropriate thread-safe context provided by the game engine.

## API Surface
The public contract is minimal, exposing only the core evaluation logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canFulfillRequirement(Store, Ref, PlayerRef) | boolean | O(1) | Evaluates if the specified player meets the reputation threshold. Returns false if the reputation group or rank asset is invalid, or if the player's reputation value is below the required minimum. |

## Integration Patterns

### Standard Usage
This class is not used directly by game logic developers. It is configured in asset files and invoked by higher-level engine systems that manage player choices. The following example illustrates how an engine system might use an instance after it has been loaded.

```java
// This is engine-level code, not typical user code.
// An instance is retrieved from a parent asset, not created directly.
ChoiceRequirement requirement = loadedChoice.getRequirement();

// The engine checks if the player meets the criteria before displaying the choice.
if (requirement.canFulfillRequirement(worldStore, playerEntityRef, playerRef)) {
    // Present the associated choice to the player.
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call new ReputationRequirement(). The asset pipeline is the sole authority for creating instances. Bypassing it will skip critical validation steps and can lead to runtime exceptions if the specified IDs are invalid.
- **State Mutation:** Do not use reflection or other means to modify the internal state after creation. The object's integrity depends on its immutability.
- **Procedural Logic:** Do not use this class to build complex, procedural requirement checks in Java code. It is designed to represent a single, data-defined rule.

## Data Pipeline
The flow of data from configuration to runtime evaluation is linear and robust.

> Flow:
> Game Asset (JSON) -> Hytale Codec System -> **ReputationRequirement** (Validated, in-memory object) -> Choice System -> canFulfillRequirement() -> Player State Lookup -> Boolean Result

