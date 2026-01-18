---
description: Architectural reference for the NonTicking component, a performance-critical singleton in the ECS framework.
---

# NonTicking

**Package:** com.hypixel.hytale.component
**Type:** Stateless Singleton Component

## Definition
```java
// Signature
public class NonTicking<ECS_TYPE> implements Component<ECS_TYPE> {
```

## Architecture & Concepts

The NonTicking component is a specialized, high-performance implementation within the Entity-Component-System (ECS) framework. Its primary function is to act as a marker, signaling to the engine's core processing loops that an entity should be excluded from per-frame update calculations, commonly known as "ticks".

Architecturally, this component embodies the **Null Object Pattern** or serves as a **Sentinel Value**. Instead of entities having a boolean flag like *isTicking* or a nullable update component, the presence of the NonTicking component itself provides this information. This design allows entity processing systems to filter entire collections of entities for ticking eligibility with a simple, type-based query, which is significantly more efficient than iterating and checking individual properties.

The implementation as a **stateless singleton** is critical for memory optimization. Since the component carries no data, there is no reason to allocate a new instance for each of the potentially thousands of non-ticking entities. A single, globally shared instance is used for all, drastically reducing the memory footprint of static or inert entities in the world.

**WARNING:** The generic parameter ECS_TYPE is a consequence of implementing the base Component interface. For this specific implementation, the type is effectively erased and ignored due to the singleton returning a single, shared instance for all specializations.

## Lifecycle & Ownership

-   **Creation:** The single instance of NonTicking is created eagerly and immediately upon class loading by the JVM, through the initialization of its `private static final` field. It is not managed by a dependency injection container or created on-demand.
-   **Scope:** Application-wide. The singleton instance persists for the entire lifetime of the application.
-   **Destruction:** The object is destroyed only when the JVM shuts down and its class loader is garbage collected. There is no manual cleanup mechanism.

## Internal State & Concurrency

-   **State:** **Immutable and Stateless**. This class contains no instance fields and its behavior is defined entirely by its type. It is a pure marker component.
-   **Thread Safety:** **Inherently thread-safe**. As a stateless object with no mutable fields, the singleton instance can be safely shared and accessed by any number of threads without locks or synchronization. The static `get` method is also safe, as it performs a simple read of a final field.

## API Surface

The public contract is minimal, designed solely to provide access to the singleton instance and satisfy the Component interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | static NonTicking | O(1) | Returns the single, shared instance of the component. This is the canonical way to acquire it. |
| clone() | Component | O(1) | Overrides the clone behavior to return the singleton instance, preventing accidental duplication. |

## Integration Patterns

### Standard Usage

The NonTicking component should be retrieved via its static `get` method and attached to an entity to prevent it from being processed by update systems.

```java
// Example: Creating a static, non-interactive rock entity
Entity rock = world.createEntity();

// Assign a model and position, which do not require ticking
rock.setComponent(new PositionComponent(100, 50, 200));
rock.setComponent(new RenderModelComponent("boulder_03"));

// Mark the entity as non-ticking for performance
// The engine's SystemProcessor will now skip this entity during its update phase
rock.setComponent(NonTicking.get());
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** The constructor is private to enforce the singleton pattern. Attempting to instantiate it via reflection or other means would violate its core design contract.
-   **Adding State:** Do not modify this class to include fields. Its performance and safety guarantees rely on it being stateless. If state is needed, a different component type is required.
-   **Null Checks:** Systems should not check if an entity's update component is null. Instead, they should query for the *absence* of the NonTicking component.

## Data Pipeline

The NonTicking component does not process data. Instead, it acts as a gate in the engine's control flow, diverting entities away from the main update pipeline.

> **Control Flow:**
> Entity System Processor -> Queries for all entities with `TickingComponent` -> **`NonTicking` entities are filtered out** -> Remaining entities are passed to `update()` logic

