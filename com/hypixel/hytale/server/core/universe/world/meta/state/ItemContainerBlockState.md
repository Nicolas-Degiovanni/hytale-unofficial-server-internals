---
description: Architectural reference for ItemContainerBlockState
---

# ItemContainerBlockState

**Package:** com.hypixel.hytale.server.core.universe.world.meta.state
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface ItemContainerBlockState {
```

## Architecture & Concepts
The ItemContainerBlockState interface defines a standardized contract for any block state that can hold items. It serves as a crucial abstraction layer, decoupling systems that interact with inventories from the concrete implementations of blocks like chests, furnaces, or custom modded containers.

This design follows the **Interface Segregation Principle**, providing a minimal, focused API for inventory access. Any system, such as player interaction handlers, automation logic (e.g., hoppers), or world serialization, can query a block for this capability without needing to know its specific type. This promotes polymorphism and greatly simplifies the logic for generic item manipulation within the world. It is the primary mechanism for exposing a block's internal inventory to the wider game engine.

## Lifecycle & Ownership
As an interface, ItemContainerBlockState does not have its own lifecycle. Instead, its lifecycle is entirely dictated by the concrete class that implements it.

- **Creation:** The implementing block state object is typically created by the world when a corresponding block is placed or when a chunk is loaded from storage.
- **Scope:** The object's lifetime is tied directly to the block it represents. It persists as long as the block exists in the world.
- **Destruction:** The implementing object is marked for garbage collection when the block is destroyed or the chunk it resides in is unloaded.

**WARNING:** References to the ItemContainer returned by this interface should not be held long-term, as the underlying block state may become invalid.

## Internal State & Concurrency
- **State:** This interface is stateless by definition. However, the implementing class and the ItemContainer it returns are highly stateful, representing the live inventory of a block.
- **Thread Safety:** The interface itself imposes no concurrency constraints. **CRITICAL:** The thread safety of the returned ItemContainer is the sole responsibility of the implementing class. Consumers of this interface must assume that direct modification of the returned container is **not thread-safe** unless explicitly documented otherwise by the implementation. All interactions with the container should be performed on the main server thread or via the world's designated task scheduler to prevent race conditions and data corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getItemContainer() | ItemContainer | O(1) | Retrieves the inventory container associated with this block state. |

## Integration Patterns

### Standard Usage
The standard pattern involves retrieving a block's state from the world, checking if it implements this interface, and then casting it to access its inventory. This is the canonical way to interact with block inventories generically.

```java
// How a developer should normally use this
BlockState state = world.getBlockState(position);

if (state instanceof ItemContainerBlockState) {
    ItemContainerBlockState containerState = (ItemContainerBlockState) state;
    ItemContainer inventory = containerState.getItemContainer();
    // Safely interact with the inventory...
    inventory.addItem(someItemStack);
}
```

### Anti-Patterns (Do NOT do this)
- **Concrete Casting:** Do not cast directly to a specific block type (e.g., ChestBlockState) to get its inventory. This defeats the purpose of the abstraction and creates brittle code that will break if the block type changes.
- **Unsafe Concurrent Modification:** Do not retrieve an ItemContainer from an asynchronous thread and modify it directly. This will lead to severe concurrency issues. All modifications must be scheduled to run on the main game tick.
- **Assuming Existence:** Do not assume a block state will implement this interface. Always perform an `instanceof` check before casting. Failure to do so will result in ClassCastExceptions.

## Data Pipeline
This interface acts as a data source, providing a gateway to a block's internal item data.

> Flow:
> System (e.g., Player Interaction Logic) -> World::getBlockState(pos) -> **ItemContainerBlockState::getItemContainer()** -> ItemContainer -> Inventory Modification Logic

