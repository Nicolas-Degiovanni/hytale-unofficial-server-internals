---
description: Architectural reference for FilterActionType
---

# FilterActionType

**Package:** com.hypixel.hytale.server.core.inventory.container.filter
**Type:** Enum (Value Object)

## Definition
```java
// Signature
public enum FilterActionType {
   ADD,
   REMOVE,
   DROP;
}
```

## Architecture & Concepts
FilterActionType is a compile-time constant enumeration that represents the set of possible actions a player or system can perform on an inventory filter. It serves as a type-safe discriminator, ensuring that any operation related to inventory filters is constrained to one of the three defined actions: ADD, REMOVE, or DROP.

Architecturally, this enum is a fundamental building block within the server-side inventory logic. It replaces primitive types like integers or strings for representing actions, which prevents a wide class of bugs related to invalid or ambiguous state. Its primary role is to provide clarity and correctness within command objects, network packets, and event payloads that manipulate inventory container filters.

## Lifecycle & Ownership
- **Creation:** The Java Virtual Machine (JVM) creates instances for ADD, REMOVE, and DROP when the FilterActionType class is first loaded by the ClassLoader. These instances are static, final, and globally accessible.
- **Scope:** These enum constants exist for the entire lifetime of the server application. They are effectively permanent singletons managed by the JVM.
- **Destruction:** The instances are garbage collected only when the ClassLoader that loaded them is itself unloaded, which typically only happens on application shutdown.

## Internal State & Concurrency
- **State:** FilterActionType instances are deeply immutable. Their state is defined at compile time and cannot be altered at runtime. They hold no dynamic data.
- **Thread Safety:** This enum is unconditionally thread-safe. As immutable singletons, its constants can be safely accessed and passed between any number of concurrent threads without synchronization. This guarantee is critical for the high-concurrency server environment.

## API Surface
The primary API consists of the constants themselves. The Java compiler also provides implicit static methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ADD | FilterActionType | O(1) | Represents the action of adding an item or rule to a filter. |
| REMOVE | FilterActionType | O(1) | Represents the action of removing an item or rule from a filter. |
| DROP | FilterActionType | O(1) | Represents the action of dropping a filtered item from an inventory. |
| values() | FilterActionType[] | O(1) | *Implicit.* Returns a cached array of all enum constants. |
| valueOf(String) | FilterActionType | O(N) | *Implicit.* Deserializes a string to its corresponding enum constant. |

## Integration Patterns

### Standard Usage
FilterActionType is intended to be used as a parameter or field within data transfer objects and service methods to direct logic flow, typically within a switch statement.

```java
// How a developer should normally use this
public void applyFilterAction(FilterActionType action, Item item) {
    switch (action) {
        case ADD:
            // Logic to add the item to the filter
            break;
        case REMOVE:
            // Logic to remove the item from the filter
            break;
        case DROP:
            // Logic to drop the item from the inventory
            break;
        default:
            throw new UnsupportedOperationException("Unknown filter action: " + action);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Ordinal-Based Logic:** Do not rely on the integer ordinal of the enum constants (e.g., `action.ordinal()`). This is extremely brittle; reordering the constants in the source file will break serialization and saved data. Always use the name for serialization.
- **Null Checks:** Methods accepting a FilterActionType should be designed to reject null values. A null action is an invalid state and should result in an immediate exception, not silent failure.

## Data Pipeline
This enum is a data component, not an active processor. It flows through the system as part of a larger command or event payload.

> Flow:
> Client Input (UI Click) -> Network Command Packet (contains FilterActionType) -> Server Command Handler -> **FilterActionType** is read -> Inventory Service -> Inventory State Change

