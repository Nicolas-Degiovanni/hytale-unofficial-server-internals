---
description: Architectural reference for DamageEntityInteraction
---

# DamageEntityInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server
**Type:** Configuration Object

## Definition
```java
// Signature
public class DamageEntityInteraction extends Interaction {
```

## Architecture & Concepts

The DamageEntityInteraction class is a data-driven, state machine node that orchestrates the entire lifecycle of a damage-dealing event within the server. It is not a service or a manager, but rather an immutable configuration asset loaded from game files. Its primary role is to serve as a declarative blueprint for how damage should be calculated, applied, and what subsequent game logic should be triggered based on the outcome.

This class acts as the central bridge between the high-level Interaction System and the low-level Damage and Entity Component Systems. It encapsulates complex combat logic, such as conditional damage based on hit location or angle, and connects to other interactions to form a directed graph of game behaviors.

A critical architectural feature is its two-phase execution model:

1.  **Calculation & Dispatch:** In the first phase, it calculates potential damage, knockback, and other effects based on its configuration. It does not apply these directly. Instead, it creates `Damage` event objects and dispatches them to the target entity via the server's `CommandBuffer`. This decouples the interaction logic from the entity processing logic.
2.  **Result Processing & Branching:** After the `Damage` events have been processed by the entity's own systems (e.g., armor calculation, damage reduction effects), this interaction is re-activated. It inspects the results of the damage events (e.g., blocked, cancelled, successful) and then jumps to the appropriate subsequent interaction defined in its `next`, `failed`, or `blocked` properties.

The `compile` method is a significant performance optimization. It pre-processes the interaction chain, creating a jump table of `Label` objects. This allows the Interaction System to branch to the next state without expensive runtime asset lookups, making combat logic highly efficient.

## Lifecycle & Ownership

-   **Creation:** Instances are deserialized from asset files by the Hytale Asset Loader using the static `CODEC` field. This process occurs during server bootstrap or when new game content is loaded. **WARNING:** Direct instantiation via the `new` keyword will result in a non-functional object that bypasses critical initialization logic.
-   **Scope:** An instance of DamageEntityInteraction is effectively a singleton for its corresponding asset. It is a shared, immutable prototype. It persists for the entire lifetime of the server.
-   **Destruction:** The object is garbage collected when the server shuts down and the Asset Loader releases its references.

## Internal State & Concurrency

-   **State:** **Immutable**. All configuration fields, such as `damageCalculator` and `angledDamage`, are set upon deserialization and are never modified at runtime. All mutable, per-execution state (e.g., the target entity, hit location) is stored externally in the `InteractionContext` object passed into the `tick0` method.
-   **Thread Safety:** **Thread-safe**. Due to its immutable nature, a single DamageEntityInteraction instance can be safely used by multiple interaction contexts across different threads without requiring locks or synchronization. The state for each execution is isolated within its respective `InteractionContext`.

## API Surface

The primary contract is with the server's Interaction System, which invokes methods based on the interaction lifecycle.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick0(...) | void | O(N) | The primary execution entry point. Dispatches or processes damage events. Complexity is O(N) where N is the number of angled/targeted damage configurations. |
| compile(...) | void | O(M) | Pre-builds the execution graph for this interaction and its children. Called once by the Interaction System. Complexity is O(M) where M is the number of chained interactions. |
| configurePacket(...) | void | O(1) | Serializes the interaction's configuration into a network packet for client-side prediction or presentation. |

## Integration Patterns

### Standard Usage

This class is not intended for direct developer invocation. It is executed exclusively by the server's Interaction System. A developer defines its behavior in an asset file (e.g., JSON), which is then loaded by the server. Another game system, such as an AI behavior tree or a player input handler, initiates the interaction by name.

```java
// PSEUDOCODE: How the engine uses this class
// This code does not exist in one place but represents the conceptual flow.

// 1. An asset file defines the interaction's properties.
//    "my_sword_attack.json" -> DamageEntityInteraction instance

// 2. A system triggers the interaction by its asset name.
Interaction interaction = Interaction.getInteraction("my_sword_attack");
InteractionContext context = new InteractionContext(attacker, target, interaction);

// 3. The InteractionSystem runs the context.
interactionSystem.run(context); // This will eventually call tick0 on the instance.
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new DamageEntityInteraction()`. The object will be uninitialized and lack a valid `CODEC`-hydrated configuration, leading to NullPointerExceptions and incorrect behavior. Always define interactions as assets.
-   **Stateful Logic:** Do not attempt to store runtime state in a subclass of DamageEntityInteraction. This violates the immutable, shared-prototype design and will cause severe concurrency issues. All state must be stored in the `InteractionContext`.
-   **Synchronous Damage Application:** Do not attempt to bypass the `CommandBuffer` and apply damage directly to an entity from within `tick0`. This breaks the server's execution model and will lead to race conditions and unpredictable outcomes.

## Data Pipeline

The flow of data and control for a single damage interaction is a multi-tick, asynchronous process orchestrated by the Interaction System.

> Flow:
> Interaction Trigger -> `InteractionSystem` creates `InteractionContext` -> `tick0` is called -> `attemptEntityDamage0` calculates damage -> `Damage` events are created -> `CommandBuffer` queues the events for the target entity -> Interaction pauses.
>
> *-- Tick Boundary --*
>
> `EntitySystem` processes its command buffer -> Target entity receives and processes `Damage` events (applies armor, etc.) -> `Damage` events are marked with results (blocked, cancelled).
>
> *-- Tick Boundary --*
>
> `InteractionSystem` resumes the context -> `tick0` is called again -> `processDamage` reads the results from the `Damage` events -> `InteractionContext` jumps to the compiled `Label` for `next`, `failed`, or `blocked` -> Execution continues with the next `Interaction` in the chain.

