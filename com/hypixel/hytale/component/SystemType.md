---
description: Architectural reference for SystemType
---

# SystemType

**Package:** com.hypixel.hytale.component
**Type:** Handle / Value Object

## Definition
```java
// Signature
public class SystemType<ECS_TYPE, T extends ISystem<ECS_TYPE>> implements Comparable<SystemType<ECS_TYPE, ?>> {
```

## Architecture & Concepts
The **SystemType** class is a foundational primitive within Hytale's Entity Component System (ECS) framework. It is not the system itself, but rather a lightweight, type-safe handle or identifier that uniquely represents a registered **ISystem** class within a specific **ComponentRegistry**.

Its primary architectural role is to provide a fast, integer-based mechanism for system lookup and management, replacing slower, reflection-based or string-based approaches. Each **SystemType** is uniquely identified by an integer index, which allows the **ComponentRegistry** to store and retrieve system instances from a flat array, yielding O(1) access patterns.

Furthermore, each **SystemType** is permanently bound to the **ComponentRegistry** that created it. This design enforces strict separation between different ECS worlds (e.g., client-side vs. server-side registries), preventing dangerous cross-contamination of systems and entities. The object acts as a "key" that only works on the "lock" that forged it.

### Lifecycle & Ownership
- **Creation:** A **SystemType** instance is created exclusively by a **ComponentRegistry** during the system registration process, typically via a method like `registerSystem`. Direct instantiation by application code is prohibited.
- **Scope:** The lifecycle of a **SystemType** is inextricably tied to its parent **ComponentRegistry**. It remains valid as long as the registry is active and has not been reset or destroyed.
- **Destruction:** The object is not explicitly destroyed but is logically invalidated. When a **ComponentRegistry** is cleared, it calls the internal `invalidate` method on all **SystemType** instances it has issued. This flips an internal flag, rendering the handle unusable. Any subsequent attempt to use an invalidated handle will result in an **IllegalStateException**, providing a critical fail-fast safety mechanism.

## Internal State & Concurrency
- **State:** The core state of a **SystemType**—its registry binding, its Java Class reference, and its unique index—is immutable after construction. The only mutable state is the `invalidated` boolean flag, which is a write-once field that transitions from false to true.
- **Thread Safety:** This class is **not thread-safe**. The state, particularly the `invalidated` flag, is not managed with atomic operations or locks. It is designed to be created, managed, and used exclusively by its owning **ComponentRegistry** within a single, well-defined thread, such as the main game loop or a dedicated world thread. Unsynchronized access from multiple threads will lead to unpredictable behavior and is a severe programming error.

## API Surface
The public API is designed for validation and identification, not mutation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getRegistry() | ComponentRegistry | O(1) | Returns the parent registry that owns this handle. |
| getTypeClass() | Class | O(1) | Returns the underlying Java Class of the **ISystem** this handle represents. |
| isType(ISystem) | boolean | O(1) | Checks if a given system instance is of the type represented by this handle. |
| getIndex() | int | O(1) | Returns the unique, zero-based integer ID for this system type within its registry. |
| validateRegistry(registry) | void | O(1) | Throws **IllegalArgumentException** if the handle is used with a foreign registry. |
| validate() | void | O(1) | Throws **IllegalStateException** if the handle has been invalidated by its parent registry. |

## Integration Patterns

### Standard Usage
Developers typically interact with system *instances*, not **SystemType** handles directly. The handle is an internal mechanism used by the ECS framework to orchestrate systems. The following example illustrates how the engine might use a **SystemType** to manage system dependencies or execution order.

```java
// Engine-level code, not typical application code
ComponentRegistry<EcsType> registry = getSomeRegistry();

// The registry creates and returns the handle during registration
SystemType<EcsType, PhysicsSystem> physicsType = registry.registerSystem(PhysicsSystem.class);

// The engine can now use this handle for fast, safe operations
// For example, to ensure physics runs before rendering
if (physicsType.getIndex() > renderingType.getIndex()) {
    throw new IllegalStateException("System dependency order is incorrect!");
}

// The handle must be validated before use, especially if cached
physicsType.validate();
```

### Anti-Patterns (Do NOT do this)
- **Caching Across Resets:** Do not cache and reuse a **SystemType** handle across a **ComponentRegistry** lifecycle (e.g., shutting down a world and starting a new one). The old handle will be invalid and will throw an exception. Always re-acquire handles from the new registry.
- **Cross-Registry Usage:** Never use a **SystemType** obtained from one registry to interact with another. This is a critical architectural violation that the `validateRegistry` method is designed to prevent.
- **Direct Instantiation:** The constructor is `protected` for a reason. Never attempt to create a **SystemType** with `new`. It will lack a proper index and registry binding, making it useless and dangerous.

## Data Pipeline
A **SystemType** is not part of the primary ECS data pipeline; it is a piece of metadata used to *control* that pipeline. It acts as a key for the **ComponentRegistry**, which then dispatches entities and components to the correct **ISystem** for processing.

> Flow:
> **ComponentRegistry** uses **SystemType** -> To look up **ISystem** instance -> **ISystem** processes Entity/Component data -> Data is mutated for the next system

