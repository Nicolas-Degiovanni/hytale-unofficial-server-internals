---
description: Architectural reference for OutOfCombatCondition
---

# OutOfCombatCondition

**Package:** com.hypixel.hytale.server.core.modules.entitystats.asset.condition
**Type:** Transient / Data-Driven

## Definition
```java
// Signature
public class OutOfCombatCondition extends Condition {
```

## Architecture & Concepts
The OutOfCombatCondition is a specific implementation of the Condition contract, designed to evaluate whether a game entity is currently in a "combat" state. It serves as a data-driven rule within the server's Entity Stats and Gameplay Ability systems.

Architecturally, this class is not a service or a manager but a stateless, configurable logic component. Its primary role is to decouple gameplay rules from hard-coded logic. Instead of writing `if (time_since_last_hit > 5_SECONDS)`, game designers can define this condition and its parameters within external asset files (e.g., JSON). The Hytale server deserializes these assets into OutOfCombatCondition instances at runtime using the provided static **CODEC**.

This class acts as a bridge between three core systems:
1.  **Asset System:** It is defined and configured entirely within game data files.
2.  **Entity Component System (ECS):** It reads the live state of an entity by querying the DamageDataComponent.
3.  **Gameplay Configuration:** It can fall back to global server settings defined in CombatConfig if a specific delay is not provided in the asset.

The static **CODEC** field is the most critical architectural feature. It is a factory and schema definition that dictates how this Java object is constructed from a serialized data format, making the condition's behavior modifiable without recompiling server code.

### Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the Hytale **Codec** framework during server startup or asset reloading. The static **CODEC** field is invoked to deserialize a data structure into a new OutOfCombatCondition object. Direct instantiation is prohibited by its protected constructor.
-   **Scope:** Extremely short-lived and transient. An instance is typically owned by a higher-level asset object, such as a stat modifier or an ability definition. It exists only as long as its parent asset is held in memory and is evaluated on-demand by the stats system.
-   **Destruction:** Managed by the Java Garbage Collector. Once the parent asset is unloaded or the evaluation that required it is complete, the instance becomes eligible for collection. There are no native resources or explicit cleanup procedures.

## Internal State & Concurrency
-   **State:** Effectively immutable. The primary state, the **delay** duration, is set once during deserialization by the **CODEC** and is never modified thereafter. The **eval0** method is a pure function with respect to the object's own state; it reads external state from the ECS but does not mutate its own fields.
-   **Thread Safety:** The object itself is thread-safe due to its immutable nature. However, the evaluation method **eval0** is **NOT** safe to call from arbitrary threads. It accesses shared, mutable ECS state via the ComponentAccessor. All evaluations must be performed on the main server thread (the world tick thread) to prevent race conditions and concurrent modification exceptions within the underlying EntityStore.

## API Surface
The public contract is fulfilled by the base **Condition** class. The core logic resides in the protected **eval0** method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval0(accessor, ref, time) | boolean | O(1) | Evaluates if the entity is out of combat. Returns true if the time since the last recorded combat action exceeds the configured delay. Throws NullPointerException if the entity's DamageDataComponent is missing. |

## Integration Patterns

### Standard Usage
A game designer or developer would not interact with this class directly in Java code. Instead, it is defined declaratively within a game asset file. The engine's stat system would then locate and evaluate this condition when needed.

*Example Asset Definition (Illustrative JSON)*
```json
{
  "id": "fast_health_regen",
  "type": "StatModifier",
  "conditions": [
    {
      "type": "OutOfCombatCondition",
      "DelaySeconds": 10.0
    }
  ],
  "effects": [
    {
      "stat": "HEALTH_REGEN",
      "value": 5.0
    }
  ]
}
```

The system responsible for applying the `fast_health_regen` modifier would automatically evaluate the OutOfCombatCondition.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use reflection or other means to create an instance. The `new OutOfCombatCondition()` pattern is invalid and bypasses the critical data-driven initialization path. All instances must originate from the **CODEC**.
-   **Asynchronous Evaluation:** Do not call the `eval` or `eval0` method from a separate thread without synchronizing on the world's main tick. Accessing ECS components from off-thread will lead to severe data corruption and server instability.

## Data Pipeline
The flow of data and logic for this component is unidirectional, starting from static configuration and resulting in a dynamic boolean check.

> Flow:
> Game Asset File (JSON) -> Server Asset Loader -> **OutOfCombatCondition.CODEC** -> In-memory **OutOfCombatCondition** instance -> Stat System evaluation trigger -> **eval0** reads **DamageDataComponent** from EntityStore -> Returns boolean result -> Gameplay logic is applied or skipped

