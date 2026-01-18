---
description: Architectural reference for the Resource interface, a core component contract in the Hytale ECS framework.
---

# Resource

**Package:** com.hypixel.hytale.component
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface Resource<ECS_TYPE> extends Cloneable {
```

## Architecture & Concepts
The Resource interface is a foundational contract within Hytale's Entity Component System (ECS). It serves as a marker interface for all data components that can be attached to an entity. It does not define behavior; rather, it establishes a component as a container for state.

In the Hytale ECS, an *entity* is merely an identifier. Its properties and data are defined by the collection of Resource components attached to it. This interface, through its generic parameter and extension of Cloneable, enforces two critical architectural principles:

1.  **Type Association:** The generic parameter `ECS_TYPE` is a placeholder for a more concrete type, allowing the ECS framework to categorize and manage components efficiently.
2.  **Mutability & Instantiation:** By extending Cloneable, the contract mandates that all components must be duplicable. This is essential for operations like entity duplication, template instantiation (prefabs), and state rollback. A Resource represents a unique instance of data for a single entity.

This interface is the primary mechanism for defining the "what" of an entity, while separate *Systems* define the "how" by operating on entities that possess specific combinations of Resources.

## Lifecycle & Ownership
As an interface, Resource itself has no lifecycle. The following describes the lifecycle of a concrete class that **implements** this interface.

-   **Creation:** A Resource instance is created when a component is added to an entity. This typically occurs during entity spawning, either from a predefined template (e.g., loading a mob from a data file) or through programmatic construction by a game system. The ECS World or a relevant manager is the sole authority for creating and attaching components.
-   **Scope:** The lifetime of a Resource instance is strictly bound to the lifetime of the entity it is attached to. It persists as long as the entity exists in the game world.
-   **Destruction:** A Resource is marked for destruction when its parent entity is destroyed or when the component is explicitly removed from the entity via the ECS API. The Java Garbage Collector is responsible for final memory reclamation. Direct manual de-allocation is not required.

## Internal State & Concurrency
The Resource interface is stateless. However, any class implementing this contract is expected to adhere to the following principles.

-   **State:** Implementations are, by design, mutable Plain Old Java Objects (POJOs). They are intended to be pure data containers whose fields are read from and written to by various game systems during the game loop tick.
-   **Thread Safety:** **WARNING:** Resource implementations are **not thread-safe** and must not be treated as such. The Hytale ECS architecture mandates that all component data is accessed and modified exclusively on the main game thread. Unsynchronized access from other threads (e.g., networking, physics) will lead to race conditions, data corruption, and engine instability. Any cross-thread data transfer must be marshaled through thread-safe queues managed by a higher-level engine service.

## API Surface
The public contract is minimal, focusing on identity and duplication.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| EMPTY_ARRAY | Resource[] | O(1) | A shared, zero-length array to prevent unnecessary allocations. |
| clone() | Resource | O(N) | Creates and returns a copy of this Resource. Complexity depends on the implementation (shallow vs. deep). |

## Integration Patterns

### Standard Usage
Developers will typically define a concrete class implementing the Resource interface to hold entity-specific data. This class is then registered with the ECS and used by game systems.

```java
// 1. Define a concrete data component
public class HealthComponent implements Resource<HealthComponent> {
    public int currentHealth = 100;
    public int maxHealth = 100;

    @Override
    public HealthComponent clone() {
        try {
            return (HealthComponent) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new AssertionError(); // Should not happen
        }
    }
}

// 2. A game system accesses the component via the ECS
public class RegenerationSystem extends GameSystem {
    public void update(World world, float deltaTime) {
        world.query(HealthComponent.class).forEach(entity -> {
            HealthComponent health = entity.getComponent(HealthComponent.class);
            if (health.currentHealth < health.maxHealth) {
                health.currentHealth += 1; // Modify state
            }
        });
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Incorrect Cloning:** Implementing a shallow copy in `clone()` for a Resource that contains object references can lead to multiple entities sharing and corrupting the same underlying data object. This is a critical and difficult-to-diagnose bug.
-   **Adding Logic:** A Resource implementation should not contain complex business logic, network calls, or file I/O. This logic belongs in a *System*. Violating this principle breaks the ECS pattern and creates highly coupled, untestable code.
-   **Cross-Thread Modification:** Never get a component from an entity and modify its fields from a worker thread. All state changes must be scheduled to run on the main game thread.

## Data Pipeline
A Resource is a payload of data that is processed by game systems. Its journey through the engine is central to the ECS data flow.

> Flow:
> Asset File (JSON/Binary) -> Deserializer -> **Concrete Resource Instance** -> Attached to a new Entity -> Queried and Mutated by Game Systems -> Serializer -> Network Packet / Save Game File

