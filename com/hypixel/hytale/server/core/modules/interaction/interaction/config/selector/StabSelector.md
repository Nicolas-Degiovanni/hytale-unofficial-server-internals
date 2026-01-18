---
description: Architectural reference for StabSelector
---

# StabSelector

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.selector
**Type:** Configuration / Factory

## Definition
```java
// Signature
public class StabSelector extends SelectorType {
```

## Architecture & Concepts
The StabSelector is a data-driven configuration class that defines a volumetric, time-progressing target selection mechanism. It represents a "stab" or "lunge" that sweeps through a volume over a set duration, detecting entities and blocks within its path.

Architecturally, this class embodies the separation of configuration from execution.
1.  **Configuration (StabSelector):** This outer class is an immutable data object, typically deserialized from a configuration file via its static CODEC. It defines the *shape* and *properties* of the selection volume: its start and end distances, its cross-sectional dimensions (extendTop, extendBottom, etc.), and rotational offsets. It acts as a factory for creating runtime instances.
2.  **Execution (RuntimeSelector):** The private inner class RuntimeSelector is the stateful, single-use object that performs the actual selection logic. Each time an interaction is initiated, a new RuntimeSelector is created. It maintains the state of the ongoing stab, calculating the sub-volume to test for the current game tick based on the total runtime of the interaction.

The core detection logic is delegated to a HitDetectionExecutor, which is configured with an OrthogonalProjectionProvider. This effectively creates a rectangular frustum (a clipped pyramid) originating from the attacker's viewpoint and extending forward. The tick method animates this frustum by progressively moving its near and far clipping planes from the configured StartDistance to the EndDistance over the interaction's duration.

## Lifecycle & Ownership
The StabSelector system has two distinct lifecycles corresponding to its configuration and execution roles. Failure to respect these lifecycles will result in undefined behavior.

-   **Creation:**
    -   The outer StabSelector is instantiated by the engine's Codec system during server asset loading. It is deserialized from a data file defining an interaction.
    -   The inner RuntimeSelector is instantiated by the interaction system by calling newSelector on a configured StabSelector instance. This must occur at the beginning of each new interaction.

-   **Scope:**
    -   A StabSelector configuration object is a shared, long-lived asset. It persists as long as its corresponding interaction configuration is loaded, typically for the entire server session.
    -   A RuntimeSelector instance is ephemeral. Its lifetime is strictly bound to a single execution of an interaction. It is created, ticked for several frames, and then must be discarded.

-   **Destruction:**
    -   StabSelector objects are garbage collected when the server unloads the assets they belong to.
    -   RuntimeSelector objects are eligible for garbage collection as soon as the interaction they service is complete. There is no explicit cleanup method.

## Internal State & Concurrency
-   **State:**
    -   **StabSelector:** Immutable. Its fields are populated once by the CODEC and are not mutated thereafter. This allows a single instance to be safely shared across the entire server.
    -   **RuntimeSelector:** Highly mutable. It maintains per-tick state, including lastTime and runTimeDeltaPercentageSum, to calculate the current position of the sweeping detection volume. It also holds stateful helper objects like HitDetectionExecutor and Matrix4d.

-   **Thread Safety:**
    -   **StabSelector:** Inherently thread-safe due to its immutability.
    -   **RuntimeSelector:** **Not thread-safe.** It is designed to be owned and operated exclusively by the thread managing the attacker's game logic (e.g., the main server tick thread for that entity). Its internal state is modified on every call to tick without any synchronization primitives. Concurrent access will lead to severe calculation errors and race conditions.

## API Surface
The public API of StabSelector is minimal, reflecting its role as a factory. The primary logic is encapsulated within the Selector interface implemented by the private RuntimeSelector.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| newSelector() | Selector | O(1) | Creates a new, stateful RuntimeSelector instance for a single interaction. |
| toPacket() | com.hypixel.hytale.protocol.Selector | O(1) | Serializes the selector's configuration into a network packet for client-side prediction or visualization. |

## Integration Patterns

### Standard Usage
A game system executing an interaction should retrieve a configured StabSelector, create a new RuntimeSelector for the interaction's duration, and tick it accordingly.

```java
// 1. Retrieve the configured selector (e.g., from an item's properties)
StabSelector stabConfig = interaction.getSelector();

// 2. At the start of the interaction, create a new runtime instance
Selector runtimeSelector = stabConfig.newSelector();

// 3. On each game tick for the duration of the interaction...
//    (time = current game time, runTime = total duration of the stab)
runtimeSelector.tick(commandBuffer, attackerRef, time, runTime);

// 4. After ticking, query for targets
runtimeSelector.selectTargetEntities(commandBuffer, attackerRef, (target, hitLocation) -> {
    // Apply damage or effects to the target
}, filter);
```

### Anti-Patterns (Do NOT do this)
-   **Runtime Instance Reuse:** Do not cache and reuse a RuntimeSelector instance across multiple, distinct interactions. Its internal time-tracking state will become invalid, leading to incorrect hit detection. Always call newSelector for each new stab.
-   **Direct Instantiation:** Do not use `new StabSelector()`. These objects are designed to be data-driven and must be created via the engine's CODEC system from configuration files.
-   **Concurrent Modification:** Do not share a RuntimeSelector instance across threads or call its tick method from a different thread than the one that owns the attacker entity.

## Data Pipeline
The StabSelector transforms an entity's state and the passage of time into a set of detected targets.

> Flow:
> Interaction Event -> Get **StabSelector** Config -> `newSelector()` -> `RuntimeSelector.tick(Attacker State)` -> Update `HitDetectionExecutor` Frustum -> `selectTargetEntities()` -> Test Against Nearby Bounding Boxes -> Invoke Consumer Callback -> Apply Game Effect

