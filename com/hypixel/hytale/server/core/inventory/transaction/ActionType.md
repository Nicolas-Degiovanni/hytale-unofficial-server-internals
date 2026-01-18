---
description: Architectural reference for ActionType
---

# ActionType

**Package:** com.hypixel.hytale.server.core.inventory.transaction
**Type:** Utility

## Definition
```java
// Signature
public enum ActionType
```

## Architecture & Concepts
ActionType is an enumeration that defines the fundamental, atomic operations that can occur within an inventory transaction. It serves as a command descriptor, encapsulating the *intent* of a modification to an inventory slot or container. This enum is a cornerstone of the server-side inventory system, providing a clear, type-safe contract for how inventory changes are processed.

Each constant (SET, ADD, REMOVE, REPLACE) is not merely a label but a composite of behavioral flags: *add*, *remove*, and *destroy*. These flags are queried by the InventoryTransactionProcessor to execute the correct logic. For example, a REPLACE action is defined as both an *add* and a *remove* operation, allowing the system to handle it atomically without needing separate transaction steps.

This design pattern prevents ambiguity in inventory logic and centralizes the definition of transactional capabilities, making the system more robust and easier to maintain.

## Lifecycle & Ownership
- **Creation:** Enum constants are instantiated by the Java Virtual Machine (JVM) during class loading. This process is guaranteed to happen only once.
- **Scope:** ActionType constants are static, final, and exist for the entire lifetime of the server application. They are effectively global singletons.
- **Destruction:** The constants are garbage collected only when the defining class loader is unloaded, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** ActionType is immutable. The internal boolean flags (*add*, *remove*, *destroy*) are private, final, and set at compile time via the constructor. The state of an ActionType constant can never change during runtime.
- **Thread Safety:** This enum is inherently thread-safe. As an immutable object with a single, globally accessible instance for each constant, it can be safely accessed and read from any thread without synchronization.

## API Surface
The public contract consists of the four enum constants and the accessor methods for their behavioral flags.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SET | ActionType | O(1) | Represents setting a slot's content, potentially destroying the original. |
| ADD | ActionType | O(1) | Represents adding an item to a slot or stack. |
| REMOVE | ActionType | O(1) | Represents removing an item from a slot or stack. |
| REPLACE | ActionType | O(1) | Represents swapping an item in a slot with another. |
| isAdd() | boolean | O(1) | Returns true if the action involves adding an item. |
| isRemove() | boolean | O(1) | Returns true if the action involves removing an item. |
| isDestroy() | boolean | O(1) | Returns true if the action overwrites and destroys the original item. |

## Integration Patterns

### Standard Usage
ActionType is intended to be used within conditional logic, typically a switch statement, to direct the flow of an inventory transaction's execution.

```java
// How a developer should normally use this
void processTransaction(InventoryTransaction tx) {
    ActionType action = tx.getActionType();

    if (action.isRemove()) {
        // Logic for removing items from the source slot
        removeItem(tx.getSourceSlot());
    }

    if (action.isAdd()) {
        // Logic for adding items to the target slot
        addItem(tx.getTargetSlot());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Enums cannot be instantiated with the *new* keyword. Attempting to do so will result in a compile-time error.
- **String or Ordinal Comparison:** Avoid comparing enums by their string representation or ordinal value. This is brittle and inefficient. Always use direct object comparison (`==`) which is safe and fast for enums.
    - **BAD:** `if (action.name().equals("ADD")) { ... }`
    - **BAD:** `if (action.ordinal() == 1) { ... }`
    - **GOOD:** `if (action == ActionType.ADD) { ... }`
- **External Logic Replication:** Do not replicate the logic of the boolean flags outside of this enum. Always query the enum directly.
    - **BAD:** `if (action == ActionType.REPLACE || action == ActionType.REMOVE) { removeItem(); }`
    - **GOOD:** `if (action.isRemove()) { removeItem(); }`

## Data Pipeline
ActionType does not process data itself; it is metadata that directs data flow within the inventory system. It is a critical component of the transaction object that travels from the point of intent to the point of execution.

> Flow:
> Player Action -> InventoryTransaction(..., **action=ActionType.ADD**) -> Server Transaction Processor -> Inventory State Modification -> Client State Update

