---
description: Architectural reference for the Option class, the core component of the NPC Utility AI system.
---

# Option

**Package:** com.hypixel.hytale.server.npc.decisionmaker.core
**Type:** Component Model / Abstract Base Class

## Definition
```java
// Signature
public abstract class Option {
```

## Architecture & Concepts
The Option class represents the fundamental unit of choice within the server-side NPC Decision-Making Subsystem. It is an abstract implementation of the Utility AI pattern, where an NPC evaluates a set of potential actions (Options) and selects the one with the highest calculated "utility" score for the current game state.

An Option is not a self-contained piece of logic. Instead, its behavior is defined by a collection of **Condition** assets. Each Condition evaluates a specific, narrow aspect of the game world—such as distance to a target, the NPC's health, or time of day—and returns a score from 0.0 to 1.0.

The core responsibility of the Option class is to aggregate the scores from its associated Conditions into a single, final utility score. This is performed within the **calculateUtility** method using a compensated product formula. This formula ensures that multiple high-scoring Conditions amplify the Option's appeal, while preventing a single moderately low-scoring Condition from disproportionately penalizing the total score. A score of exactly 0.0 from any Condition, however, will immediately disqualify the Option.

A key performance optimization is the pre-sorting of Conditions by their computational complexity via the **sortConditions** method. This allows the utility calculation to fail-fast, evaluating the cheapest Conditions first. If a cheap Condition returns a score of 0.0, the more expensive evaluations for that Option are skipped entirely for the current tick.

## Lifecycle & Ownership
- **Creation:** Option instances are **never** instantiated directly via code using the *new* keyword. They are deserialized from server-side asset files (e.g., JSON) by the Hytale codec system during server startup or asset hot-reloading. The static **ABSTRACT_CODEC** field defines the schema for this data-driven instantiation.

- **Scope:** An Option is a stateless configuration asset. Once loaded, it is held in a global asset registry and persists for the entire server session. The same Option instance is shared and referenced by all NPC entities whose behavior is governed by it.

- **Destruction:** The object is eligible for garbage collection only when the server's Asset Manager unloads its corresponding asset file, typically during a server shutdown or a full asset reload.

## Internal State & Concurrency
- **State:** The state of an Option is considered immutable after the initial loading and initialization phase. Fields such as *conditions* and *weightCoefficient* are populated from asset files. The *sortedConditions* array is a derived, cached representation of the *conditions* and is populated by a single call to **sortConditions**.

- **Thread Safety:** This class is **not thread-safe** and must not be accessed concurrently. All interactions, particularly calls to **calculateUtility**, are designed to occur exclusively on the main server thread that processes the NPC update loop. The internal **ConditionReference** class utilizes a WeakReference as a memory-management strategy, allowing Condition assets to be garbage collected during hot-reloads without causing memory leaks. The reference is robustly re-acquired from the global asset map if it has been cleared.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| sortConditions() | void | O(N log N) | **CRITICAL INITIALIZER.** Populates the internal condition cache, sorting by computational simplicity. Must be called once after asset loading and before any utility calculations. |
| setupNPC(Role role) | void | O(N) | Propagates entity-specific setup logic to all child Conditions. Called when an NPC with this Option is initialized. |
| calculateUtility(...) | double | O(N) | Calculates the final utility score for this Option based on the current game state. Returns 0.0 if the Option is not viable. |

## Integration Patterns

### Standard Usage
An Option is evaluated by a higher-level system, such as an **Evaluator**, which manages the decision-making tick for an NPC. The Evaluator retrieves the relevant Options for an NPC's current state and iterates through them to find the best one.

```java
// Pseudocode for a hypothetical Evaluator system
// This code does NOT exist in the Option class itself.

// During NPC initialization
for (Option option : npc.getBehaviorProfile().getOptions()) {
    option.sortConditions(); // Must be called once
}

// During an NPC's update tick
Option bestOption = null;
double maxUtility = 0.0;

for (Option option : npc.getBehaviorProfile().getOptions()) {
    double utility = option.calculateUtility(self, chunk, target, buffer, context);
    if (utility > maxUtility) {
        maxUtility = utility;
        bestOption = option;
    }
}

// Execute the behavior associated with bestOption
if (bestOption != null) {
    // ...
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never attempt to create a concrete subclass of Option with `new`. All Options must be defined as data in asset files and loaded by the server.
- **Skipping Initialization:** Calling **calculateUtility** before **sortConditions** has been executed will result in a NullPointerException and a server crash.
- **Runtime Modification:** Do not modify the state of an Option object after it has been loaded. These are shared, stateless assets. Modifying one will affect every NPC that uses it, leading to unpredictable behavior.
- **Concurrent Access:** Do not read or execute an Option from any thread other than the primary NPC processing thread.

## Data Pipeline
The flow of data from configuration to execution is a one-way process managed by the engine.

> Flow:
> NPC Behavior Asset File (JSON) -> Server Asset Loader -> **Option Deserialization (via ABSTRACT_CODEC)** -> In-Memory Asset Registry -> NPC Evaluator -> **calculateUtility()** -> Utility Score -> NPC Action

