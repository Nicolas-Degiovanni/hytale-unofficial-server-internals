---
description: Architectural reference for CraftingManager
---

# CraftingManager

**Package:** com.hypixel.hytale.builtin.crafting.component
**Type:** Transient (ECS Component)

## Definition
```java
// Signature
public class CraftingManager implements Component<EntityStore> {
```

## Architecture & Concepts
The CraftingManager is a stateful ECS component that manages a player's active crafting session with a specific workbench or crafting interface. It is not a global singleton; instead, an instance is typically attached to a Player entity when they interact with a crafting station. This component acts as the central state machine for all crafting-related operations for that player, including timed crafting, bench upgrades, and inventory transactions.

Its primary responsibilities are:
- **Session Management:** It maintains a reference to the specific bench block in the world (via coordinates) that the player is currently using. This session begins with a call to setBench and ends with clearBench.
- **Job Queuing:** It manages a queue of time-based crafting tasks using an internal BlockingQueue. This decouples the immediate request to craft from the time-based execution, which is processed by the game's main tick loop.
- **State Progression:** The tick method serves as the component's heartbeat, advancing the progress of active crafting jobs and bench upgrades frame by frame.
- **Transaction Logic:** It orchestrates the complex process of consuming input materials from a player's inventory and granting the resulting output items, ensuring atomicity for each step of a recipe.

Architecturally, CraftingManager is the bridge between the player's inventory system, the world's block state (for the bench), and the server's core game loop.

## Lifecycle & Ownership
- **Creation:** A CraftingManager component is added to a Player entity by the server logic, typically in response to the player interacting with a crafting bench block in the world. The default constructor creates a dormant instance; it is only activated upon a successful call to the setBench method.
- **Scope:** The component's active state persists for the duration of a single crafting session. This session is defined as the time between the player opening a crafting bench UI and closing it. All queued jobs and state are tied to this session.
- **Destruction:** The session is terminated by calling clearBench. This method is critical for cleanup, as it cancels all pending jobs, refunds materials for incomplete tasks, and resets the component's internal state. The component itself may persist on the entity in a dormant state, ready for a new session.

## Internal State & Concurrency
- **State:** The CraftingManager is highly mutable. Its core state includes the coordinates of the active bench, the type of bench block, a queue of CraftingJob objects, and an optional BenchUpgradingJob. This state represents the complete context of the player's current crafting activity.
- **Thread Safety:** The component is conditionally thread-safe. The use of a LinkedBlockingQueue for queuedCraftingJobs allows producer threads (e.g., network packet handlers calling queueCraft) to safely add jobs without conflicting with the consumer thread (the main game loop calling tick). However, direct access to other fields or mutation of a job's state outside of the tick method is not safe and will lead to undefined behavior. All state progression is designed to be handled synchronously within the tick method.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setBench(x, y, z, blockType) | void | O(1) | Initializes the manager for a session with a specific bench. Throws IllegalArgumentException if a session is already active. |
| clearBench(ref, store) | boolean | O(N) | Terminates the current session, cancels all jobs, and resets state. N is the number of queued jobs. |
| craftItem(ref, store, recipe, quantity, container) | boolean | O(M) | Executes an instantaneous craft. Performs validation and inventory transactions immediately. M is the number of material types. |
| queueCraft(ref, store, window, id, recipe, quantity, container, type) | boolean | O(1) | Enqueues a time-based crafting job. Does not block. Returns false if the bench is invalid for the recipe or is currently upgrading. |
| tick(ref, store, dt) | void | O(1) | Advances the state of the active crafting or upgrade job. Designed to be called once per game loop tick. |
| startTierUpgrade(ref, store, window) | boolean | O(1) | Initiates the bench upgrade process if requirements are met. Prevents new crafting jobs from starting. |
| cancelAllCrafting(ref, store) | boolean | O(N) | Drains the crafting queue and attempts to refund materials for the first job. N is the number of queued jobs. |

## Integration Patterns

### Standard Usage
The CraftingManager is exclusively managed by server-side systems in response to player actions. A typical flow involves the server opening a BenchWindow for the player, which then drives the lifecycle of this component.

```java
// Context: A player has just interacted with a crafting bench block.
// The server retrieves the player's entity and the CraftingManager component.

CraftingManager manager = store.getComponent(playerRef, CraftingManager.getComponentType());

// 1. Initialize the session with the bench's location and type.
BlockType benchBlockType = world.getBlockTypeAt(x, y, z);
manager.setBench(x, y, z, benchBlockType);

// 2. Player requests to craft an item via the UI, sending a packet.
// The packet handler calls queueCraft on the player's manager.
manager.queueCraft(playerRef, store, playerWindow, transactionId, recipe, 1, playerInventory, InputRemovalType.NORMAL);

// 3. The server's main loop calls tick() on the component every frame,
// which processes the job queue.

// 4. When the player closes the UI, the server must clean up the session.
manager.clearBench(playerRef, store);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use new CraftingManager(). As an ECS component, it must be retrieved from the EntityStore using a ComponentAccessor to ensure it is correctly associated with an entity.
- **State Machine Violation:** Do not call setBench on a manager that is already associated with a bench. You must call clearBench first to terminate the existing session. Failure to do so will result in an IllegalArgumentException.
- **External Queue Modification:** Do not directly access or modify the internal queuedCraftingJobs collection. Use the provided API methods like queueCraft and cancelAllCrafting to ensure state consistency.
- **Skipping Cleanup:** Forgetting to call clearBench when the player's crafting session ends (e.g., they close the UI, die, or disconnect) will leave the component in a stale state and may prevent future crafting sessions from starting correctly.

## Data Pipeline
The data flow for a standard, time-based crafting operation demonstrates the component's role as a stateful processor operating between network input and the game loop.

> Flow:
> Player Input (UI) -> Client Network Packet -> Server Packet Handler -> **CraftingManager.queueCraft()** -> Internal Job Queue -> **CraftingManager.tick()** [Game Loop] -> Inventory Transaction -> Player Inventory Update -> Client State Sync

