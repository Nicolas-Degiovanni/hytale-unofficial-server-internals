---
description: Architectural reference for the Flag interface, a core component for bitmasking operations.
---

# Flag

**Package:** com.hypixel.hytale.common.collection
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface Flag {
```

## Architecture & Concepts
The Flag interface defines a contract for objects, typically enums, that represent individual bits within a bitmask. This pattern is a foundational performance optimization used throughout the engine for managing sets of states, permissions, or options in a memory-efficient manner.

Instead of using collections like a HashSet to store active states, the engine can combine multiple Flag implementors into a single integer or long value. This allows for extremely fast set-based operations (add, remove, check for presence) using bitwise arithmetic. The interface guarantees that any object conforming to this contract can provide a unique integer bitmask, enabling generic systems like a FlagSet to operate on them without knowledge of the concrete type.

This component is fundamental to systems requiring high-performance state management, such as entity component states, network packet options, or rendering flags.

## Lifecycle & Ownership
As an interface, Flag itself has no lifecycle. The lifecycle and ownership semantics apply to the classes that *implement* this interface.

- **Creation:** Implementors are almost exclusively enums. Enum constants are instantiated by the JVM during class loading. They are effectively static singletons.
- **Scope:** The scope of a Flag implementation is global. Once the implementing enum class is loaded, its constants persist for the entire lifetime of the application.
- **Destruction:** Flag constants are unloaded and garbage collected only when their defining ClassLoader is garbage collected, which for core engine components, means upon application shutdown.

## Internal State & Concurrency
- **State:** The Flag interface defines no state. By convention and for system stability, all implementing classes **must be immutable**. The values returned by name() and mask() must be constant for the lifetime of the object. Enums naturally satisfy this requirement.
- **Thread Safety:** All implementations of Flag are expected to be inherently thread-safe. Because the intended implementors are enums, their constants are singletons with final fields, making them safe to access from any thread without synchronization.

**WARNING:** Creating a mutable class that implements Flag is a severe violation of the contract and will lead to unpredictable behavior, race conditions, and state corruption in systems that rely on bitmasking.

## API Surface
The public contract is minimal, designed to provide the necessary components for bitmasking operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| name() | String | O(1) | Returns the programmatic name of the flag. Primarily used for debugging, logging, and serialization. |
| mask() | int | O(1) | Returns the integer bitmask for this specific flag. This value **must** be a power of two (e.g., 1, 2, 4, 8). |

## Integration Patterns

### Standard Usage
The standard and expected pattern is to implement the Flag interface on an enum. Each enum constant represents a distinct option and is assigned a unique, power-of-two bitmask.

```java
// How a developer should normally use this
public enum RenderFlag implements Flag {
    IS_VISIBLE(1),
    HAS_SHADOW(2),
    IS_TRANSLUCENT(4);

    private final int mask;

    RenderFlag(int mask) {
        this.mask = mask;
    }

    @Override
    public int mask() {
        return this.mask;
    }
}

// This enum can then be used with a FlagSet or for direct bitwise checks.
int entityFlags = RenderFlag.IS_VISIBLE.mask() | RenderFlag.HAS_SHADOW.mask();
boolean isVisible = (entityFlags & RenderFlag.IS_VISIBLE.mask()) != 0; // true
```

### Anti-Patterns (Do NOT do this)
- **Non-Power-of-Two Masks:** Assigning masks that are not a power of two (e.g., `HAS_SHADOW(3)`) will cause bit collisions and break the logic of any system using the flags, as multiple flags would map to the same bits.
- **Mutable Implementations:** Creating a standard class that implements Flag where the mask value can change at runtime is strictly forbidden. This violates the core assumption of stability and will corrupt any bitmask it is added to.
- **Zero Mask:** Assigning a mask of 0 is invalid, as it represents the absence of any flag and cannot be uniquely identified in a bitwise AND operation.

## Data Pipeline
The Flag interface is not part of a data processing pipeline itself, but rather a tool for creating and interpreting data. It enables the transformation of high-level concepts into a low-level, efficient data format.

> Conceptual Flow:
> High-Level Enum Constant (e.g., RenderFlag.IS_VISIBLE) -> **Flag.mask()** -> Integer Bit (e.g., 1) -> Combined into Bitmask Field (e.g., an entity's state integer) -> Bitwise Check for Presence

