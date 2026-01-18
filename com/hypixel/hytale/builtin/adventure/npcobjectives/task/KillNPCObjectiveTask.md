---
description: Architectural reference for KillNPCObjectiveTask
---

# KillNPCObjectiveTask

**Package:** com.hypixel.hytale.builtin.adventure.npcobjectives.task
**Type:** Transient / Data Object

## Definition
```java
// Signature
public class KillNPCObjectiveTask extends KillObjectiveTask {
```

## Architecture & Concepts
The KillNPCObjectiveTask is a concrete implementation within the Adventure Mode objective framework. It represents a specific, quantifiable goal: a player or entity must kill a target number of a specific NPC type. This class is a specialization of the more generic KillObjectiveTask, tailored for NPC entities.

Architecturally, this class does not actively track game events. Instead, it functions as a **setup and configuration component**. Its primary responsibility is to bridge the high-level Objective system with the low-level entity event system during objective activation. It achieves this by creating and registering a transaction listener (KillTaskTransaction) with the appropriate resource (KillTrackerResource) on the entity responsible for the objective.

This design decouples the objective definition from the runtime tracking mechanism. The KillNPCObjectiveTask holds the *intent* of the objective, while the actual stateful tracking is delegated to resources and transactions within Hytale's Entity Component System (ECS). The static CODEC field indicates that this class is designed for serialization, allowing objective progress and definitions to be persisted in save games or transmitted over the network.

### Lifecycle & Ownership
-   **Creation:** Instances are created by the server-side Objective system when an Objective containing this task is activated for an entity. It is configured using a KillObjectiveTaskAsset, which defines the target NPC and the required kill count. Instances can also be created via deserialization from storage or network packets, facilitated by the public static CODEC.
-   **Scope:** The object's lifetime is strictly bound to its parent Objective. It persists as long as the objective is active for a given entity.
-   **Destruction:** The instance is eligible for garbage collection once the parent Objective is completed, failed, or otherwise removed from an entity. The objective management system is responsible for using the TransactionRecord returned by the setup method to unregister the associated listener, preventing memory leaks and redundant processing.

## Internal State & Concurrency
-   **State:** This class is effectively **immutable** after construction. It holds configuration data (the asset, task index) but does not store mutable runtime state, such as the current kill count. All runtime progress is managed externally by the KillTaskTransaction and KillTrackerResource. This separation of configuration from state is a critical design choice for system stability.
-   **Thread Safety:** The class is inherently thread-safe for read operations due to its immutable nature. However, the setup0 method is a state-changing operation on the world and is **not thread-safe**. It is designed to be called exclusively from the main server thread during the objective activation phase. Concurrent calls to setup0 for the same objective would result in duplicate listeners and severe logic errors.

## API Surface
The public contract is minimal, centered entirely on the setup process.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup0(objective, world, store) | TransactionRecord[] | O(1) | Initializes the kill tracking logic. Creates and registers a transaction to listen for kill events on the target entity's KillTrackerResource. **Warning:** This method is not idempotent and must only be called once per objective lifecycle. |

## Integration Patterns

### Standard Usage
A developer will not typically interact with this class directly. It is managed by the higher-level objective system. The following example illustrates the conceptual engine-level usage during objective activation.

```java
// Conceptual example of engine code activating the task
Objective parentObjective = ...;
Store<EntityStore> playerEntityStore = ...;
World world = ...;

// The task is retrieved from the objective's definition
KillNPCObjectiveTask task = parentObjective.getTask(taskIndex);

// The engine calls setup0 to wire up the event listeners
TransactionRecord[] records = task.setup0(parentObjective, world, playerEntityStore);

// The engine stores these records to manage teardown later
parentObjective.setActiveTransactions(records);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not manually construct this class with `new`. Objective tasks should always be defined via data assets and managed by the objective framework.
-   **Reusing Instances:** Do not attempt to reuse a task instance across different objectives or entities. Each task is tied to a specific context provided during the `setup0` call.
-   **Multiple Setup Calls:** Calling `setup0` more than once on the same task instance will register duplicate listeners on the `KillTrackerResource`. This will cause every subsequent kill to be counted multiple times, breaking objective logic.

## Data Pipeline
The KillNPCObjectiveTask initiates a data flow but does not participate in the per-event processing. It sets up the pipeline, which then runs independently.

> Flow:
> Objective Activation -> Engine calls **KillNPCObjectiveTask.setup0()** -> A `KillTaskTransaction` is created -> The transaction registers itself with an entity's `KillTrackerResource`
>
> *Later, during gameplay...*
>
> NPC Death Event -> `KillTrackerResource` is notified -> `KillTrackerResource` invokes the registered `KillTaskTransaction` -> The transaction updates the parent `Objective`'s progress -> The `Objective` system checks for completion.

