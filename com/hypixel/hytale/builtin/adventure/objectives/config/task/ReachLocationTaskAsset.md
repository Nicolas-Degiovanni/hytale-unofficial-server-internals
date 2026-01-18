---
description: Architectural reference for ReachLocationTaskAsset
---

# ReachLocationTaskAsset

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.task
**Type:** Configuration Asset

## Definition
```java
// Signature
public class ReachLocationTaskAsset extends ObjectiveTaskAsset {
```

## Architecture & Concepts
The ReachLocationTaskAsset is a data-driven configuration object, not an active runtime component. It serves as a static data template that defines a specific type of objective within Hytale's Adventure Mode framework: requiring a player to reach a designated location.

Its primary architectural feature is the static **CODEC** field. This object integrates the class into the engine's core asset serialization and deserialization pipeline. The game engine uses this codec to parse adventure mode data files (e.g., JSON) and hydrate this strongly-typed Java object. This pattern ensures that all objective definitions are loaded, validated, and represented in a consistent, type-safe manner before they are used by the game's Objective System.

A critical design choice is the inclusion of validation logic directly within the codec definition. The codec ensures that the **targetLocationId** is not just a string, but a valid reference to an existing **ReachLocationMarkerAsset**. This preemptively enforces data integrity at asset load time, preventing runtime errors caused by misconfigured or missing location markers.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the Hytale Asset Loading system when parsing adventure mode content packs. The static **CODEC** is invoked to construct the object from a serialized data source. Manual instantiation is an anti-pattern and will result in an incomplete object.
-   **Scope:** An instance of this asset is immutable and persists in memory for the lifetime of its parent content pack. It is treated as a read-only definition shared across the server or client.
-   **Destruction:** The object is garbage collected when the server or client unloads the associated content pack. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
-   **State:** Immutable. After instantiation by the asset pipeline, the internal state, specifically the **targetLocationId**, is never modified. All methods are pure functions that read this state.
-   **Thread Safety:** This class is inherently thread-safe due to its immutability. A single instance can be safely accessed and read by multiple game systems (e.g., AI, Player Logic, UI) across different threads without requiring any synchronization mechanisms.

## API Surface
The public API is minimal, designed for read-only access to the objective's configuration data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getTaskScope() | ObjectiveTaskAsset.TaskScope | O(1) | Returns the hardcoded scope, **PLAYER**, indicating this task is tracked on a per-player basis. |
| getTargetLocationId() | String | O(1) | Retrieves the unique identifier for the target **ReachLocationMarkerAsset**. |

## Integration Patterns

### Standard Usage
This class is not intended to be used procedurally. Its primary integration point is through declarative data files that define adventure mode content. The game's Objective System consumes these loaded assets to manage player state.

A system interacting with a player's active objectives might encounter this asset as follows:
```java
// Example of a system consuming the loaded asset
ActiveObjective objective = player.getObjectiveManager().getActiveObjective();
ObjectiveTaskAsset taskAsset = objective.getCurrentTaskAsset();

if (taskAsset instanceof ReachLocationTaskAsset reachLocationTask) {
    String locationId = reachLocationTask.getTargetLocationId();
    // Use locationId to find the marker in the world and update UI or game logic
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use **new ReachLocationTaskAsset()**. The object will lack its **targetLocationId** and will fail validation checks in any system that consumes it. Assets must be loaded via the engine's asset pipeline.
-   **State Mutation:** Do not use reflection or other means to modify the **targetLocationId** field after creation. The Objective System relies on the immutability of this asset for predictable behavior and thread safety.

## Data Pipeline
The ReachLocationTaskAsset exists as part of the engine's configuration and asset loading pipeline. It transforms declarative data on disk into a usable, in-memory object for the runtime.

> Flow:
> Adventure Mode JSON File -> Engine Asset Loader -> **ReachLocationTaskAsset.CODEC** (Deserialization & Validation) -> In-Memory **ReachLocationTaskAsset** Instance -> Objective System (Runtime Consumer)

