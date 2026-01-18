---
description: Architectural reference for Flags
---

# Flags<T extends Flag>

**Package:** com.hypixel.hytale.common.collection
**Type:** Utility / Value Object

## Definition
```java
// Signature
public class Flags<T extends Flag> {
```

## Architecture & Concepts
The Flags class is a high-performance, memory-efficient utility for managing a collection of boolean states using a single integer as a bitmask. It provides a type-safe, object-oriented abstraction over low-level bitwise operations.

This component is fundamental for systems requiring the tracking of numerous binary properties, such as entity states, rendering options, or network packet attributes. By consolidating multiple booleans into one integer, it significantly reduces the memory footprint of objects compared to using individual boolean fields.

The generic constraint `T extends Flag` is the core of its type-safety mechanism. Any enum or class used with Flags must implement the `Flag` contract, which mandates a `mask()` method. This method provides the unique integer bit (e.g., 1, 2, 4, 8) for each flag, abstracting away the raw integer values from the developer and preventing common bitmasking errors.

## Lifecycle & Ownership
- **Creation:** Instances are created directly via the `new` keyword. This class is not a managed service and is not intended for dependency injection. It is a value object, owned entirely by the class that instantiates it.
- **Scope:** The lifecycle of a Flags object is bound to its owner. It is typically used as a member field and persists as long as its parent object exists.
- **Destruction:** Cleanup is handled by the Java Garbage Collector when the owning object is no longer referenced. No manual de-allocation is necessary.

## Internal State & Concurrency
- **State:** The core state is a single, mutable `int` field named `flags`. The object's identity is defined by the value of this integer. All state-modifying methods like `set` and `toggle` directly mutate this field.

- **Thread Safety:** **This class is not thread-safe.** The bitwise operations used for modification (e.g., `this.flags = this.flags | flag.mask()`) are not atomic. If a single Flags instance is accessed and modified by multiple threads concurrently without external synchronization, it will lead to race conditions and unpredictable state corruption.

    **WARNING:** All access to a shared Flags instance from multiple threads **must** be protected by locks or other concurrency control mechanisms.

## API Surface
The public API provides a clean, declarative interface for bitmask manipulation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| is(T flag) | boolean | O(1) | Atomically checks if a specific flag is currently set. |
| not(T flag) | boolean | O(1) | Atomically checks if a specific flag is *not* set. |
| set(T flag, boolean value) | boolean | O(1) | Sets or unsets a flag. Returns true if the internal state was changed. **Not thread-safe.** |
| toggle(T flag) | boolean | O(1) | Inverts the state of a flag. Returns the new state of the flag. **Not thread-safe.** |
| getFlags() | int | O(1) | Returns the raw underlying integer bitmask. Use primarily for serialization or interop with low-level systems. |

## Integration Patterns

### Standard Usage
The Flags class should be used as a member field to manage the internal state of an object. The generic type should be an enum that implements the `Flag` contract.

```java
// Assume a RenderFlag enum exists that implements the Flag contract.
// public enum RenderFlag implements Flag { ... }

// In a class like a renderer or entity:
private final Flags<RenderFlag> renderFlags;

public MyComponent() {
    // Initialize with a default set of flags
    this.renderFlags = new Flags<>(RenderFlag.IS_VISIBLE, RenderFlag.CASTS_SHADOWS);
}

public void update() {
    // Modify state
    this.renderFlags.set(RenderFlag.IS_DIRTY, true);

    // Check state
    if (this.renderFlags.is(RenderFlag.CASTS_SHADOWS)) {
        // Perform shadow calculation...
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Shared Mutable State:** Never share a single Flags instance across multiple threads for modification without explicit, external locking. This is the most common and severe misuse of this class.
- **Over-reliance on getFlags():** Avoid retrieving the raw integer with `getFlags()` for direct bitwise manipulation in application logic. Doing so bypasses the type safety provided by the class and creates a brittle dependency on its internal implementation. Reserve its use for serialization or performance-critical, low-level engine code.

## Data Pipeline
The Flags class is not a pipeline component but a terminal state container. It acts as a destination for state-change commands and a source for state queries.

> Flow:
> Application Logic -> **Flags.set() / toggle()** -> Internal `int` State Update -> **Flags.is()** -> Conditional Branch in Application Logic

