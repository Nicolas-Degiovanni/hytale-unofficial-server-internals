---
description: Architectural reference for LivingEntityInventoryChangeEvent
---

# LivingEntityInventoryChangeEvent

**Package:** com.hypixel.hytale.server.core.event.events.entity
**Type:** Transient

## Definition
```java
// Signature
public class LivingEntityInventoryChangeEvent extends EntityEvent<LivingEntity, String> {
```

## Architecture & Concepts
The LivingEntityInventoryChangeEvent is a specific, immutable event object used within the server's core event-driven architecture. It functions as a data-carrying message that signals a state change within an inventory system.

This class extends the generic EntityEvent, establishing a direct link between the event and the LivingEntity that owns the inventory. Its primary role is to decouple the inventory management subsystem from other game systems. Instead of directly querying inventory state, systems like networking, AI, or quest logic can subscribe to this event. This pattern ensures that dependent systems are notified reactively, promoting low coupling and high cohesion in the server architecture.

When an item is added, removed, or moved within an ItemContainer belonging to a LivingEntity, the inventory system commits a Transaction and then instantiates and dispatches this event to the central event bus.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the server's inventory management system immediately after a Transaction is successfully applied to an ItemContainer. It is a direct result of a state-altering operation.
- **Scope:** Ephemeral. The object's lifetime is bound to its dispatch cycle on the event bus. It exists only for the brief period during which it is passed to and processed by all registered event listeners.
- **Destruction:** The object is eligible for garbage collection as soon as the event bus finishes dispatching it to all subscribers. There is no manual memory management or cleanup required.

## Internal State & Concurrency
- **State:** Effectively immutable. The class contains no public setters, and its fields (itemContainer, transaction) are set only once during construction. It serves as a snapshot of the change at a specific moment in time. The contained objects themselves might be mutable, but from the perspective of the event consumer, they should be treated as read-only data.
- **Thread Safety:** This class is inherently thread-safe due to its immutable nature. It can be safely passed between threads, for example, from the main server tick thread to a separate networking or logging thread.

**WARNING:** While the event object itself is safe, the objects it references (ItemContainer, Transaction, LivingEntity) are part of the live game state. Listeners operating in different threads must implement their own synchronization mechanisms when accessing or modifying the underlying game state to prevent race conditions.

## API Surface
The public contract is minimal, providing read-only access to the event's context.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getEntity() | LivingEntity | O(1) | Inherited from EntityEvent. Returns the entity whose inventory changed. |
| getItemContainer() | ItemContainer | O(1) | Returns the specific inventory container that was modified. |
| getTransaction() | Transaction | O(1) | Returns the transaction object detailing the exact nature of the change. |

## Integration Patterns

### Standard Usage
The intended use is within a listener or handler class that is registered with the server's event bus. The handler receives the event object and uses its data to trigger further logic.

```java
// Example of a listener reacting to the event
public class PlayerNetworkSyncListener {

    @EventHandler
    public void onInventoryChange(LivingEntityInventoryChangeEvent event) {
        // Only act on player entities
        if (event.getEntity() instanceof Player) {
            Player player = (Player) event.getEntity();
            Transaction transaction = event.getTransaction();

            // Create a network packet based on the transaction
            Packet inventoryUpdatePacket = createPacketFromTransaction(transaction);

            // Send the update to the specific player's client
            player.getConnection().send(inventoryUpdatePacket);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Instantiation:** Do not create instances of LivingEntityInventoryChangeEvent manually using its constructor. Firing a synthetic event can lead to severe state desynchronization between the server's authoritative inventory and what other systems (or clients) perceive. These events must only originate from the core inventory system.
- **State Modification:** Do not modify the state of the ItemContainer or Transaction objects retrieved from the event. The event represents a historical fact—something that has already happened. Attempting to alter these objects from within a listener constitutes a dangerous side effect that can corrupt game state.

## Data Pipeline
This event is a critical link in the data flow between the core game logic and peripheral systems.

> Flow:
> Player Action (e.g., item pickup) -> Inventory System Logic -> Transaction Commit -> **LivingEntityInventoryChangeEvent** (Instantiation) -> Server Event Bus -> Subscribed Listeners (Network Synchronizer, Quest Logic, Logging Service)

