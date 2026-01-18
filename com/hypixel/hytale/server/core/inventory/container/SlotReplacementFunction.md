---
description: Architectural reference for SlotReplacementFunction
---

# SlotReplacementFunction

**Package:** com.hypixel.hytale.server.core.inventory.container
**Type:** Functional Interface

## Definition
```java
// Signature
public interface SlotReplacementFunction {
   ItemStack replace(short var1, ItemStack var2);
}
```

## Architecture & Concepts
The SlotReplacementFunction interface is a core component of the server's inventory system, embodying the **Strategy Pattern** for slot-specific behaviors. It defines a single, stateless contract for the logic that executes when an item is placed into or swapped with an inventory slot.

Instead of embedding complex conditional logic within a generic Container or Slot class, the system delegates the replacement behavior to an implementation of this interface. This design decouples the fundamental mechanics of an inventory (a collection of slots) from the specialized rules governing each slot. For example, a furnace's fuel slot can have a different SlotReplacementFunction than its output slot, allowing one to accept only burnable items and the other to forbid player placement entirely.

This component is fundamental to creating modular, reusable, and easily configurable inventory UIs and their corresponding server-side logic.

### Lifecycle & Ownership
- **Creation:** Implementations are typically provided as lambdas or method references during the construction of a server-side Container. The system responsible for defining a container's layout and rules will instantiate these functions. They are not managed services and should not be retrieved from a central registry.
- **Scope:** The lifecycle of a SlotReplacementFunction instance is tightly bound to the Container or Slot that holds a reference to it. It is a short-lived behavioral object, not a persistent service.
- **Destruction:** The function is eligible for garbage collection as soon as its owning Container is destroyed. No manual cleanup is necessary.

## Internal State & Concurrency
- **State:** As an interface, SlotReplacementFunction is inherently stateless. Implementations are **strongly expected to be stateless**. They should operate exclusively on the arguments provided and the state of the inventory they are modifying.
- **Thread Safety:** The interface itself is thread-safe. However, implementations operate on shared inventory state and are **not inherently thread-safe**. All inventory modifications must be synchronized by the calling thread, which is typically the main server thread responsible for a given world or dimension. Implementations must not introduce their own locking or asynchronous operations.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| replace(short slotIndex, ItemStack newItem) | ItemStack | O(1) (Typical) | Executes the replacement logic for a given slot. Returns the ItemStack that was displaced or the remainder of the newItem stack if the insertion was partial. A null return indicates the slot was empty and the new item was fully accepted. |

## Integration Patterns

### Standard Usage
A SlotReplacementFunction is provided during the definition of a container's behavior, often when adding a specific slot to its layout. The implementation defines the rules for that slot.

```java
// Hypothetical example of a Container builder
ContainerBuilder builder = new ContainerBuilder();

// A standard slot that just swaps items
builder.addSlot(0, (slotIndex, newItem) -> {
    ItemStack oldItem = inventory.getItem(slotIndex);
    inventory.setItem(slotIndex, newItem);
    return oldItem;
});

// A "deny-all" slot, like a crafting output
builder.addSlot(1, (slotIndex, newItem) -> {
    // Return the item that was attempted to be placed, signifying failure
    return newItem;
});
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** An implementation must not contain its own mutable state. Logic should be self-contained and deterministic based on the provided inputs. Storing state can lead to unpredictable behavior, especially if the same function instance is reused.
- **Blocking Operations:** The replace method is executed synchronously on a critical server thread. Performing network requests, file I/O, or any other long-running task within this function will cause severe server performance degradation.
- **Complex Object Graph:** The function should not hold strong references to heavy objects or services, as this can complicate garbage collection and create memory leaks if the owning container's lifecycle is not managed correctly.

## Data Pipeline
This function acts as a rule-based gate within the server's inventory update pipeline. It does not source or sink data but rather transforms it according to game logic.

> Flow:
> Client Input (Network Packet) -> Server Interaction Handler -> Container.onSlotClicked() -> **SlotReplacementFunction.replace()** -> Inventory State Mutation -> State Sync (Network Packet) -> Client UI Update

