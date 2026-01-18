---
description: Architectural reference for SensorWithEntityFilters
---

# SensorWithEntityFilters

**Package:** com.hypixel.hytale.server.npc.corecomponents
**Type:** Framework Component / Abstract Base Class

## Definition
```java
// Signature
public abstract class SensorWithEntityFilters extends SensorBase implements IAnnotatedComponentCollection {
```

## Architecture & Concepts
SensorWithEntityFilters is an abstract base class that forms a foundational component within the NPC Artificial Intelligence perception system. It enhances the basic `SensorBase` by introducing a powerful and reusable entity filtering pipeline. Its primary architectural purpose is to decouple the act of *detecting* potential entities from the logic of *validating* them as legitimate targets.

This class employs a **Composite Design Pattern** to manage a collection of `IEntityFilter` components. It treats the entire collection of filters as a single unit, propagating lifecycle events (such as `spawned`, `loaded`, `unloaded`) to each filter it contains. This ensures that all sub-components are correctly initialized and torn down in synchronization with the parent NPC.

The core filtering mechanism operates as a **Chain of Responsibility**. When the `matchesFilters` method is invoked, a target entity is passed sequentially through each `IEntityFilter`. If any filter in the chain rejects the entity, the process halts immediately, and the entity is deemed invalid. An entity is only considered a match if it successfully passes every filter in the collection.

This design promotes high cohesion and low coupling. Concrete sensor implementations (e.g., a sensor that finds entities in a radius) can focus solely on efficient spatial queries, while complex validation logic (e.g., checking for faction, line-of-sight, or status effects) can be encapsulated in discrete, reusable `IEntityFilter` classes.

## Lifecycle & Ownership
-   **Creation:** This abstract class is never instantiated directly. It is extended by a concrete sensor implementation. The instance is created during the construction of an NPC's `Role` component, typically via a builder pattern that assembles the required `IEntityFilter` array and passes it to the `super` constructor.
-   **Scope:** The lifecycle of a SensorWithEntityFilters instance is strictly bound to its parent NPC's `Role`. It persists as long as the NPC is active in the world.
-   **Destruction:** The object is eligible for garbage collection when the parent NPC is removed from the world. The `unloaded` and `removed` methods serve as the final lifecycle callbacks, which are cascaded down to all contained filters for their own cleanup.

## Internal State & Concurrency
-   **State:** The primary internal state is the `filters` array of type `IEntityFilter`. This array is marked `final` and is assigned only once during construction. The class does not modify this collection post-construction, making its own state effectively immutable.
-   **Thread Safety:** This class is inherently thread-safe as its own state is immutable. However, the overall thread safety of an instance depends entirely on the implementations of the `IEntityFilter` objects it contains. The Hytale server's NPC and entity systems are designed to be accessed by a single world thread. Calls from other threads are not supported and will lead to undefined behavior, including concurrency exceptions.

## API Surface
The public API consists primarily of lifecycle event handlers delegated from the parent `Role`. The most critical method is the `protected` `matchesFilters`, which is intended for use by subclasses.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matchesFilters(ref, targetRef, role, store) | protected boolean | O(N) | Evaluates a target entity against the filter chain. N is the number of filters. Returns false on the first filter failure. |
| registerWithSupport(role) | void | O(N) | Cascades the `registerWithSupport` lifecycle event to all contained filters. |
| loaded(role) | void | O(N) | Cascades the `loaded` lifecycle event to all contained filters. |
| spawned(role) | void | O(N) | Cascades the `spawned` lifecycle event to all contained filters. |
| unloaded(role) | void | O(N) | Cascades the `unloaded` lifecycle event to all contained filters. |
| removed(role) | void | O(N) | Cascades the `removed` lifecycle event to all contained filters. |

## Integration Patterns

### Standard Usage
A developer must extend this class to create a concrete sensor. The subclass is responsible for implementing the logic to find potential entities and then using `matchesFilters` to validate them.

```java
// A concrete sensor implementation that finds hostile players
public class HostilePlayerSensor extends SensorWithEntityFilters {

    public HostilePlayerSensor(BuilderSensorBase builder, IEntityFilter[] customFilters) {
        // Combine standard filters with any custom ones
        IEntityFilter[] allFilters = new IEntityFilter[] {
            new FactionFilter(Faction.HOSTILE),
            new EntityTypeFilter(EntityType.PLAYER),
            new LineOfSightFilter()
        };
        // The super constructor is critical for initializing the filter chain
        super(builder, allFilters);
    }

    // In the sensor's update/tick method...
    public void findTargets(Role role, Store<EntityStore> store) {
        // 1. Subclass performs a broad search for nearby entities
        List<Ref<EntityStore>> nearbyEntities = findEntitiesInRadius(role.getNpcRef(), 32.0);

        // 2. Subclass uses the inherited method to validate each entity
        for (Ref<EntityStore> target : nearbyEntities) {
            if (matchesFilters(role.getNpcRef(), target, role, store)) {
                // This is a valid target, add to memory or engage
                role.getAiMemory().addPotentialTarget(target);
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Bypassing the Filter Chain:** Do not manually iterate the filters or implement custom filtering logic within a subclass. The purpose of this class is to provide a centralized, ordered filtering pipeline via `matchesFilters`. Bypassing it leads to duplicated logic and breaks the chain-of-responsibility contract.
-   **Mutable Filter State:** Do not pass a mutable array to the constructor that is later modified by external code. The filter collection is assumed to be immutable after construction. Modifying it at runtime can lead to severe and hard-to-debug concurrency issues.
-   **Incorrect Super Constructor Call:** Subclasses must call `super(builder, filters)` with a valid, non-null array of filters. Failure to do so will leave the sensor in an invalid state and result in a NullPointerException.

## Data Pipeline
The data flow for this component is a clear, linear pipeline for entity validation. The subclass acts as the data source, and this base class acts as the processing pipeline.

> Flow:
> Concrete Sensor Subclass (e.g., `ProximitySensor`) -> Finds raw `Ref<EntityStore>` -> **SensorWithEntityFilters.matchesFilters** -> `IEntityFilter[0].matchesEntity()` -> `IEntityFilter[1].matchesEntity()` -> ... -> Final Boolean Result (Is Valid Target)

