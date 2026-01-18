---
description: Architectural reference for MoveType
---

# MoveType

**Package:** com.hypixel.hytale.server.core.inventory.transaction
**Type:** Enumeration

## Definition
```java
// Signature
public enum MoveType {
```

## Architecture & Concepts
MoveType is a fundamental, type-safe enumeration that represents the direction of an item movement within the context of a single inventory transaction. It provides a clear and unambiguous contract for inventory operations, eliminating the ambiguity of using booleans or integers to represent directionality.

This enumeration is a core component of the inventory system's transaction logic. It allows higher-level systems, such as InventoryTransactionProcessor, to implement distinct logic paths for when items are added to an inventory versus when they are removed. Its primary role is to act as a static data descriptor, not as an active service.

## Lifecycle & Ownership
- **Creation:** Enum constants are instantiated automatically by the Java Virtual Machine (JVM) when the MoveType class is first loaded. This process is handled entirely by the JVM's class loading mechanism.
- **Scope:** The instances MOVE_TO_SELF and MOVE_FROM_SELF are static, final singletons that persist for the entire lifetime of the application. There will only ever be one instance of each constant.
- **Destruction:** The enum instances are garbage collected only when the defining class loader is unloaded, which typically occurs at JVM shutdown. For all practical purposes, they exist forever.

## Internal State & Concurrency
- **State:** MoveType is deeply immutable. Its state is defined at compile time and cannot be altered at runtime. It holds no dynamic data or cache.
- **Thread Safety:** As an immutable, static construct, MoveType is inherently thread-safe. It can be safely accessed and passed between any number of threads without synchronization.

## API Surface
The primary API consists of the enum constants themselves, which are used for assignment and comparison.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| MOVE_TO_SELF | MoveType | O(1) | Represents a transaction where items are moved *into* the target inventory. |
| MOVE_FROM_SELF | MoveType | O(1) | Represents a transaction where items are moved *out of* the target inventory. |

## Integration Patterns

### Standard Usage
MoveType is used as a parameter to methods or as a field within data transfer objects to specify the intent of an inventory operation.

```java
// A hypothetical transaction processor using the MoveType
public void processTransaction(Inventory target, ItemStack items, MoveType direction) {
    if (direction == MoveType.MOVE_TO_SELF) {
        // Logic for adding items to the inventory
        target.add(items);
    } else if (direction == MoveType.MOVE_FROM_SELF) {
        // Logic for removing items from the inventory
        target.remove(items);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Ordinal Comparison:** Do not rely on the `ordinal()` method for logical comparisons. The order of enum declarations can change, leading to fragile and unreadable code. Always compare instances directly.

```java
// BAD: Prone to breaking if enum order changes
if (direction.ordinal() == 0) { /* ... */ }

// GOOD: Robust and self-documenting
if (direction == MoveType.MOVE_TO_SELF) { /* ... */ }
```

- **String Comparison:** Avoid converting the enum to a string for comparison. This is inefficient and bypasses the benefits of type safety.

```java
// BAD: Inefficient and not type-safe
if (direction.toString().equals("MOVE_TO_SELF")) { /* ... */ }
```

## Data Pipeline
MoveType does not process data itself; it is a piece of data that flows through other systems to direct logic.

> Flow:
> User Input (e.g., drag-and-drop item) -> UI Event -> InventoryTransactionRequest (contains **MoveType**) -> Server Network Handler -> InventoryTransactionProcessor -> Inventory State Change

