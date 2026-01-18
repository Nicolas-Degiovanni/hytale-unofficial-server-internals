---
description: Architectural reference for StateOption
---

# StateOption

**Package:** com.hypixel.hytale.server.npc.decisionmaker.stateevaluator
**Type:** Transient Data Model

## Definition
```java
// Signature
public class StateOption extends Option {
```

## Architecture & Concepts
The StateOption class is a specialized, data-driven implementation of the abstract Option class. Its primary role is to represent a potential state transition for a server-side NPC within the Decision Maker framework. It acts as a declarative data structure that defines a target state and an optional sub-state.

This class is designed to be defined in external configuration files, such as JSON behavior packs, and deserialized at runtime by the Hytale Codec system. This pattern decouples high-level NPC behavior design (e.g., "enter combat state") from the low-level engine implementation.

Architecturally, it employs a two-phase initialization model. It is first deserialized with human-readable string identifiers for states. Subsequently, a resolver process within the NPC system populates the integer-based `stateIndex` and `subStateIndex` fields. This provides the benefit of human-readable configuration while enabling high-performance, index-based lookups during the per-tick decision-making loop.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the static `StateOption.CODEC` during the deserialization of an NPC's behavior configuration. The constructor is protected to prevent direct, manual instantiation, which is a critical design constraint.

-   **Scope:** The lifetime of a StateOption object is bound to its parent configuration component, typically a `StateEvaluator`. It persists as long as the NPC's behavior definition is loaded in memory.

-   **Destruction:** The object is marked for garbage collection when its parent configuration is unloaded. This typically occurs when an NPC entity is removed or when server assets are reloaded.

## Internal State & Concurrency
-   **State:** The object's state is mutable during a controlled, two-phase initialization.
    1.  **Phase 1 (Deserialization):** The `state` and `subState` string fields are populated from the configuration source. The index fields are uninitialized (zero).
    2.  **Phase 2 (Resolution):** The NPC system calls `setStateIndex` to populate the integer indices based on the string names.
    After Phase 2, the object must be treated as immutable. Any modification will lead to desynchronization and undefined behavior.

-   **Thread Safety:** This class is **not thread-safe**. It is designed to be created, configured, and resolved on a single thread, typically the main server thread or a world-loading thread. During gameplay, its data should only be read by the NPC's decision-making logic. Writing to this object from multiple threads is an unsupported and dangerous operation.

## API Surface
The public API is primarily for read-only access to state data after initialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getState() | String | O(1) | Returns the human-readable name of the target state. |
| getSubState() | String | O(1) | Returns the optional human-readable name of the target sub-state. |
| getStateIndex() | int | O(1) | Returns the resolved, high-performance integer index for the state. |
| getSubStateIndex() | int | O(1) | Returns the resolved, high-performance integer index for the sub-state. |
| setStateIndex(stateIndex, subStateIndex) | void | O(1) | **Internal Engine Use Only.** Populates the integer indices. |

## Integration Patterns

### Standard Usage
This object is not intended to be used directly by gameplay logic. It is a component consumed by the NPC Decision Maker system. The system evaluates a list of options and, upon selecting a StateOption, uses its resolved indices to command the NPC's state machine.

```java
// Conceptual example of how the engine uses a resolved StateOption.
// This code would exist within a core NPC system, not typical mod or script code.

StateOption chosenOption = stateEvaluator.getBestOption(npcContext);

// The state machine uses the pre-calculated index for a fast, direct transition
// without requiring any string comparisons or hash map lookups.
npc.getStateMachine().transitionTo(
    chosenOption.getStateIndex(),
    chosenOption.getSubStateIndex()
);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never attempt to create an instance with `new StateOption()`. The protected constructor enforces that all instances must be created via the `CODEC` system from a valid data source.
-   **Manual Index Management:** Do not call `setStateIndex` from gameplay code. This method is part of the engine's internal asset resolution pipeline. Calling it manually can corrupt the NPC's behavior by desynchronizing the state name from its runtime index.
-   **Runtime Modification:** Do not modify the `state` or `subState` fields (e.g., via reflection) after the NPC has been initialized. The resolved indices will become invalid, leading to unpredictable state transitions or server crashes.

## Data Pipeline
The StateOption serves as a critical data container that transforms declarative configuration into a performance-optimized runtime object.

> Flow:
> NPC Behavior File (JSON) -> Codec Deserializer -> **StateOption Instance** (with string names) -> NPC Index Resolver -> **StateOption Instance** (with integer indices) -> State Evaluator -> NPC State Machine

---

