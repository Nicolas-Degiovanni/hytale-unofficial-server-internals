---
description: Architectural reference for ProcessingBenchState
---

# ProcessingBenchState

**Package:** com.hypixel.hytale.builtin.crafting.state
**Type:** State Object

## Definition
```java
// Signature
public class ProcessingBenchState
   extends BenchState
   implements TickableBlockState,
   ItemContainerBlockState,
   DestroyableBlockState,
   MarkerBlockState,
   PlacedByBlockState {
```

## Architecture & Concepts

ProcessingBenchState is a stateful component that represents the complete runtime state of a single processing-style crafting bench within the world, such as a furnace, smelter, or alloy forge. It is not a global manager but rather a self-contained state machine, instantiated for each corresponding block placed in the game world.

Its architecture is defined by the interfaces it implements, which integrate it directly into various engine subsystems:
*   **TickableBlockState:** This is the most critical integration. It registers the state object with the world's main game loop, which invokes the `tick` method on a regular basis. This method drives all core logic: consuming fuel, advancing recipe progress, and producing output items.
*   **ItemContainerBlockState:** This interface signifies that the block can hold items. The state manages three distinct internal inventories: `inputContainer`, `fuelContainer`, and `outputContainer`. It exposes a unified view of these via a `CombinedItemContainer`.
*   **DestroyableBlockState:** Provides a hook, `onDestroy`, that is invoked by the world system immediately before the block is removed. This ensures graceful cleanup, primarily the ejection of all contained items into the world as physical entities.
*   **MarkerBlockState & PlacedByBlockState:** These interfaces connect the block to the world map and player tracking systems, allowing it to have a map icon and execute special logic upon being placed by a player.

The component's primary responsibility is to manage the lifecycle of a crafting process. It continuously evaluates its input inventory to find a matching `CraftingRecipe`. Once a valid recipe is identified, and fuel is available (if required), it enters a processing state, updating its progress until the recipe is complete. It then consumes the input ingredients and places the resulting items into its output inventory.

This class also acts as a communication hub between the server-side state and any players interacting with the bench. It maintains a map of open `ProcessingBenchWindow` instances, pushing state changes (e.g., progress updates, fuel level) to connected clients to update their user interface in real-time.

## Lifecycle & Ownership

-   **Creation:** A ProcessingBenchState instance is created and hydrated by the world's chunk loading system. The static `CODEC` field defines how the object is deserialized from persistent world storage. It is never instantiated directly in game logic. The `initialize` method is called post-deserialization to link the state to its corresponding `BlockType` asset and configure its inventories.
-   **Scope:** The object's lifetime is strictly coupled to the physical block it represents in the world. It persists as long as the block exists.
-   **Destruction:** The instance is marked for garbage collection when the associated block is destroyed. The engine invokes the `onDestroy` method immediately before destruction, which serves as the final opportunity for cleanup. This includes closing all associated UI windows and dropping its inventory into the world.

## Internal State & Concurrency

-   **State:** This object is highly mutable and stateful. Its core state includes the contents of its three `ItemContainer`s, the current `inputProgress` (float), remaining `fuelTime` (float), the active `CraftingRecipe`, and a boolean `active` flag. This state is persisted to disk via the provided `CODEC`.
-   **Thread Safety:** This component is not inherently thread-safe, but is designed to operate within a specific threading model. The `tick` method is expected to be called exclusively from the main world thread. However, player interactions, such as opening the bench's UI or toggling its active state, may originate from network threads. The use of a `ConcurrentHashMap` for `windows` indicates a design to safely handle UI registration from multiple threads. All world-mutating operations, such as spawning entities, are correctly dispatched to the main thread via `world.execute`.

    **Warning:** Direct modification of this object's state from any thread other than the main world thread will lead to race conditions, data corruption, and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, ...) | void | O(C) | The primary logic driver, called by the game engine. Advances fuel consumption and recipe progress. Complexity is constant relative to the number of benches. |
| setActive(boolean) | boolean | O(W) | Toggles the processing state of the bench. Notifies all open windows (W) of the change. |
| getItemContainer() | CombinedItemContainer | O(1) | Returns a unified, read-only view of the input, fuel, and output inventories. |
| onDestroy() | void | O(N) | Cleans up the state, dropping all contained items (N) into the world. Called by the engine. |
| getRecipe() | CraftingRecipe | O(1) | Returns the currently active crafting recipe, or null if no valid recipe is matched. |

## Integration Patterns

### Standard Usage

Developers typically do not interact with this class directly. Logic that needs to inspect a bench's state should retrieve it from the world system using coordinates.

```java
// Hypothetical example of retrieving state from the world
BlockState state = world.getBlockStateAt(position);

if (state instanceof ProcessingBenchState bench) {
    // Check if the bench is currently active
    boolean isActive = bench.isActive();

    // Safely interact with the inventory
    CombinedItemContainer container = bench.getItemContainer();
    // ... inspect container contents
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new ProcessingBenchState()`. The world serialization system is the sole owner of its lifecycle. Manual creation will result in a disconnected, non-functional object.
-   **Manual Ticking:** Do not call the `tick` method manually. This will break the engine's timing and can cause unpredictable behavior, such as accelerated or duplicated processing.
-   **Direct Field Modification:** Modifying fields like `inputProgress` or `fuelTime` directly bypasses essential logic and synchronization with client UIs. Use methods like `setActive` to manipulate state.
-   **Unsafe Inventory Access:** Modifying the internal item containers directly from another thread is not safe. All interactions should be routed through the game's inventory transaction system.

## Data Pipeline

The ProcessingBenchState acts as a processor in two primary data flows.

**Crafting Process Flow:**
> Flow:
> World Tick -> **ProcessingBenchState.tick()** -> Evaluate Recipe -> Consume Fuel & Ingredients -> Advance `inputProgress` -> Create Output `ItemStack` -> Place in `outputContainer`

**UI Synchronization Flow:**
> Flow:
> State Change (e.g., `inputProgress` update) -> **ProcessingBenchState** -> `sendProgress()` -> `ProcessingBenchWindow` -> Network Packet -> Client UI Render

