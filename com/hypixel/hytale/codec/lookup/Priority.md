---
description: Architectural reference for Priority
---

# Priority

**Package:** com.hypixel.hytale.codec.lookup
**Type:** Value Object / Transient

## Definition
```java
// Signature
public class Priority {
```

## Architecture & Concepts
The Priority class is a fundamental value object used throughout the engine's codec and event systems to establish a deterministic order of operations. It encapsulates a simple integer level, providing a clear and type-safe mechanism for defining precedence in processing pipelines, listener chains, or any system implementing a chain-of-responsibility pattern.

Architecturally, Priority decouples components from hard-coded execution orders. Instead of explicitly calling components A, then B, then C, developers register components with an associated Priority. A central manager or pipeline builder then sorts and invokes these components based on their priority level. This promotes modularity and allows for the injection of new behaviors into a processing chain without modifying the core logic.

The class provides two static, well-known instances, **NORMAL** and **DEFAULT**, which serve as common baselines for relative ordering. The fluent API, with methods like `before()` and `after()`, encourages developers to define priorities relative to these known constants rather than using arbitrary "magic numbers".

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand whenever a priority needs to be specified. This can be through direct instantiation with `new Priority(level)` or, more commonly, by deriving a new Priority from an existing one via the `before()` and `after()` methods. The static instances **NORMAL** and **DEFAULT** are created once at class-loading time.
- **Scope:** The lifetime of a Priority object is transient. It is a lightweight value object that is typically held as a member of a configuration or registration object. Its scope is tied to the object that references it.
- **Destruction:** Instances are managed by the Java Garbage Collector and are destroyed when they are no longer referenced. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The Priority class is **immutable**. Its internal `level` field is private and set only once during construction. All methods that appear to modify the priority, such as `before()` and `after()`, do not alter the instance's state; they return a *new* Priority object with the calculated level.
- **Thread Safety:** Due to its immutable design, the Priority class is inherently **thread-safe**. Instances can be safely shared and accessed across multiple threads without any need for external synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| **DEFAULT** | static Priority | O(1) | A predefined, low-priority level (-1000). |
| **NORMAL** | static Priority | O(1) | A predefined, neutral priority level (0). |
| getLevel() | int | O(1) | Returns the raw integer value of the priority level. |
| before() | Priority | O(1) | Returns a new Priority instance with a level one less than the current. |
| before(int by) | Priority | O(1) | Returns a new Priority instance with a level decreased by the specified amount. |
| after() | Priority | O(1) | **WARNING:** Returns a new Priority with a level one *less* than the current. See Anti-Patterns. |
| after(int by) | Priority | O(1) | **WARNING:** Returns a new Priority with a level *decreased* by the specified amount. See Anti-Patterns. |

## Integration Patterns

### Standard Usage
Priority is used to configure the execution order of registered handlers or listeners. The common pattern is to use a predefined priority as a base and derive a new one from it.

```java
// How a developer should normally use this
CodecRegistry registry = getCodecRegistry();

// Register a primary handler at the normal level
registry.register(new PrimaryCodec(), Priority.NORMAL);

// Register a fallback handler to run after the normal one
// Note the use of the 'before' method due to the implementation bug
registry.register(new FallbackCodec(), Priority.NORMAL.before(10));

// Register a high-priority override to run before the normal one
registry.register(new OverrideCodec(), Priority.NORMAL.after(10)); // This will be higher priority
```

### Anti-Patterns (Do NOT do this)
- **Misinterpreting `after()`:** The `after(int by)` method contains a critical implementation flaw. It subtracts from the level, identical to `before(int by)`. In this system, a lower integer value means a *later* execution. Therefore, `after()` actually creates a *higher* priority, and `before()` creates a *lower* priority. This is counter-intuitive and a significant source of bugs.
  - **BAD:** `myPriority.after()` expecting it to run later.
  - **GOOD:** `myPriority.before()` to run later.
  - **GOOD:** `myPriority.after()` to run earlier.

- **Ignoring Immutability:** Believing that methods like `before()` modify the object in-place. This will lead to incorrect priority registrations.
  - **BAD:** `Priority p = Priority.NORMAL; p.before(); registry.register(handler, p);` // Registers with NORMAL priority
  - **GOOD:** `Priority p = Priority.NORMAL.before(); registry.register(handler, p);` // Registers with a new, lower priority

## Data Pipeline
The Priority class itself does not process data. Instead, it is metadata that dictates the structure of a data pipeline. It is consumed by a builder or registry component during an initialization phase.

> Flow:
> Handler Registration (with **Priority** object) -> PipelineBuilder sorts all handlers by `priority.getLevel()` -> Data Packet arrives -> Pipeline executes handlers in the sorted order -> Processed Data Egresses

