---
description: Architectural reference for ActionInventory
---

# ActionInventory

**Package:** com.hypixel.hytale.server.npc.corecomponents.items
**Type:** Transient

## Definition
```java
// Signature
public class ActionInventory extends ActionBase {
```

## Architecture & Concepts

The ActionInventory class is a concrete implementation of the `ActionBase` contract, representing a single, discrete operation that an NPC can perform on an entity's inventory. It functions as a command object within the server's NPC behavior system. Its primary role is to bridge the high-level declarative AI configuration (e.g., an NPC's role defined in an asset file) with the low-level, imperative game state systems, specifically the `Inventory` and `EntityStore` components.

This class is designed to be configured and instantiated once during the asset loading phase via its corresponding builder, `BuilderActionInventory`. This follows a data-driven design pattern where complex NPC behaviors are assembled from a palette of pre-defined, configurable actions.

An `ActionInventory` instance encapsulates a specific inventory manipulation, such as adding, removing, or equipping an item. The `Operation` enum defines the full set of possible manipulations, making this a versatile and reusable component for a wide range of NPC behaviors, from a merchant restocking its wares to a guard equipping a weapon upon detecting a threat.

## Lifecycle & Ownership
-   **Creation:** An `ActionInventory` is not created dynamically during gameplay. It is instantiated by the NPC asset loading pipeline, specifically through a `BuilderActionInventory` which parses configuration from an NPC definition file. This occurs when the server loads or reloads its game assets.

-   **Scope:** The object is immutable and stateless. Its lifetime is tied to the `Role` object that contains it. It persists as long as the NPC's behavioral definition is held in memory. The same instance can be executed repeatedly without side effects on the action object itself.

-   **Destruction:** The object is eligible for garbage collection when its parent `Role` is unloaded. This typically happens during a server shutdown or a full asset reload. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** The internal state of `ActionInventory` is **immutable**. All configuration fields (`operation`, `item`, `count`, etc.) are marked as `final` and are set exclusively at construction time. The class does not cache data or change its own state during the `execute` method. It acts as a pure data-carrier for the action's parameters.

-   **Thread Safety:** This class is not thread-safe for execution. While its immutable state makes it safe to *read* from any thread, the `execute` method performs write operations on shared, mutable game state (specifically, an entity's `Inventory` via the `EntityStore`).

    **WARNING:** The `execute` and `canExecute` methods must be called exclusively from the main server thread that manages the game world tick. Invoking these methods from an asynchronous task or a different thread will lead to race conditions, data corruption, and server instability. The system relies on the server's single-threaded game loop for synchronization.

## API Surface

The public contract is defined by its parent, `ActionBase`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Performs precondition checks to determine if the action can run. Verifies that a target exists (if required) and that an item is specified for operations that need one. |
| execute(...) | boolean | O(1) | Executes the configured inventory operation on the target entity. Resolves the entity reference, retrieves its inventory, and applies the change. Returns false if the target entity cannot be found. |

## Integration Patterns

### Standard Usage

This class is not intended for direct instantiation or invocation by gameplay programmers. It is a component of the NPC AI framework, configured via asset files and executed by an NPC's `Role` during the server tick. The system's internal logic is responsible for invoking the action.

```java
// Conceptual example of how the NPC's Role might invoke the action.
// Developers do not write this code; it is part of the core AI loop.

// Assume 'currentAction' is an ActionInventory instance loaded from an asset.
if (currentAction.canExecute(npcRef, role, sensorInfo, dt, store)) {
    boolean success = currentAction.execute(npcRef, role, sensorInfo, dt, store);
    // ... handle success or failure
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new ActionInventory()`. The object is complex to configure and must be created via the `BuilderActionInventory` during the server's asset loading phase to ensure its parameters are correctly resolved.

-   **Asynchronous Execution:** Do not call `execute` from a separate thread or asynchronous task. All modifications to game state must occur on the main server thread to prevent concurrency issues.

-   **Stateful Subclassing:** Avoid extending this class to add mutable state. Actions are designed to be reusable, stateless commands. Adding state breaks this contract and can lead to unpredictable behavior when the same action is executed multiple times.

## Data Pipeline

The flow of data from configuration to execution is a key aspect of the NPC system.

> Flow:
> NPC Behavior Asset (JSON/Config) -> `BuilderActionInventory` -> **ActionInventory Instance** (held by `Role`) -> NPC AI Engine (Decision) -> `execute()` call -> `EntityStore` Lookup -> `Inventory` Modification -> Game State Change -> Client Synchronization

