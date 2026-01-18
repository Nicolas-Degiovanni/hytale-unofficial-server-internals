---
description: Architectural reference for EmptyItemContainer
---

# EmptyItemContainer

**Package:** com.hypixel.hytale.server.core.inventory.container
**Type:** Singleton

## Definition
```java
// Signature
public class EmptyItemContainer extends ItemContainer {
```

## Architecture & Concepts
The EmptyItemContainer is a specific implementation of the **Null Object Pattern** for the inventory system. Its primary architectural purpose is to provide a safe, non-null, and behaviorally inert representation of an inventory for entities or game objects that do not possess one.

By offering a singleton instance that conforms to the ItemContainer contract, the system can avoid pervasive null checks and the risk of NullPointerExceptions. All operations that attempt to modify or query the contents of this container are designed to be either no-ops (doing nothing) or to fail explicitly by throwing an UnsupportedOperationException, enforcing its zero-capacity nature.

This design is also a significant performance optimization. Instead of allocating a new, empty inventory object for every entity that lacks one, the entire server shares a single, immutable `INSTANCE`. This drastically reduces memory allocation and garbage collection overhead.

## Lifecycle & Ownership
- **Creation:** The singleton `INSTANCE` is instantiated statically when the EmptyItemContainer class is first loaded by the Java Virtual Machine. Its lifecycle is not tied to any specific game state or session.
- **Scope:** Application-wide. The `INSTANCE` is a global constant that persists for the entire lifetime of the server process.
- **Destruction:** The object is never explicitly destroyed. It is eligible for garbage collection only when its class loader is unloaded, which typically occurs at server shutdown.

## Internal State & Concurrency
- **State:** **Immutable**. The EmptyItemContainer contains no instance fields and its state cannot be modified. Its behavior is constant and predictable. Methods that would typically return dynamic data, such as `toProtocolMap`, instead return pre-allocated, empty collections.
- **Thread Safety:** This class is inherently thread-safe due to its complete immutability. The singleton `INSTANCE` can be safely shared and accessed by any number of threads without requiring locks or any other synchronization mechanisms. The `readAction` and `writeAction` methods bypass any locking logic present in the parent class, as there is no mutable state to protect.

## API Surface
The public contract of EmptyItemContainer is designed to fulfill the ItemContainer interface while preventing any state changes.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getCapacity() | short | O(1) | Always returns 0. This is the primary method for identifying this container's nature. |
| clear() | ClearTransaction | O(1) | A no-op that returns the static `ClearTransaction.EMPTY` instance. |
| forEach(action) | void | O(1) | A no-op. The provided action is never executed as there are no items to iterate over. |
| clone() | EmptyItemContainer | O(1) | Returns the singleton `INSTANCE` itself, reinforcing the singleton pattern. |
| registerChangeEvent(p, c) | EventRegistration | O(1) | Returns a static, shared, no-op event registration. No events are ever fired. |
| internal_getSlot(slot) | ItemStack | O(1) | **CRITICAL:** Always throws UnsupportedOperationException. |
| internal_setSlot(slot, item) | ItemStack | O(1) | **CRITICAL:** Always throws UnsupportedOperationException. |

## Integration Patterns

### Standard Usage
The primary use case is to assign this container to entities or components that require an ItemContainer but have no actual inventory. This satisfies API contracts without special-casing or nulls.

```java
// Assign the singleton instance to an entity that has no inventory
ItemContainer container = EmptyItemContainer.INSTANCE;

// This check will always be false, safely preventing modification attempts
if (container.getCapacity() > 0) {
    // This code is unreachable
    container.setSlot((short) 0, new ItemStack(...));
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is protected. Never attempt to create a new instance. Always use the static `EmptyItemContainer.INSTANCE`.
- **Type Checking:** Avoid checks like `if (container instanceof EmptyItemContainer)`. The system is designed to rely on behavior, not type. Use `if (container.getCapacity() == 0)` to determine if an inventory can hold items. This respects the Liskov Substitution Principle and makes the code more robust.
- **Unguarded Modification:** Do not call methods like `setSlot` or `getSlot` without first checking the capacity. While a standard ItemContainer might return null, this implementation will throw a runtime exception, which can crash server logic if not handled.

## Data Pipeline
The EmptyItemContainer acts as a terminal node, or a "sink," in any data pipeline involving item manipulation. It does not hold, transform, or forward data. Any attempt to push item data into it is effectively voided.

> Flow:
> Item Transfer Logic -> **EmptyItemContainer** -> No-op or UnsupportedOperationException (Operation Terminated)

