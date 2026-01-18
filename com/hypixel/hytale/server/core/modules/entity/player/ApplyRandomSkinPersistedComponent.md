---
description: Architectural reference for ApplyRandomSkinPersistedComponent
---

# ApplyRandomSkinPersistedComponent

**Package:** com.hypixel.hytale.server.core.modules.entity.player
**Type:** Singleton

## Definition
```java
// Signature
public class ApplyRandomSkinPersistedComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The ApplyRandomSkinPersistedComponent is a server-side **Tag Component** within the Hytale Entity-Component-System (ECS) framework. Unlike typical components that hold data, this class is stateless and acts as a marker or a flag. Its presence on an entity signals to other systems that a specific one-time action needs to be performed: assigning a random skin to the entity.

Its primary role is to trigger behavior within a separate processing system, such as a PlayerSkinSystem. The component itself contains no logic. The "Persisted" suffix in its name, combined with the presence of a `CODEC`, indicates that this flag is designed to be saved with the entity's data. This ensures that the skin randomization logic can be correctly applied even after a server restart, for an entity that has not yet had its skin assigned.

As a singleton, a single `INSTANCE` is shared across all entities that have this component attached. This design is highly memory-efficient, as it avoids allocating a new object for every entity that needs this tag.

## Lifecycle & Ownership
- **Creation:** The singleton `INSTANCE` is instantiated once by the JVM during class loading. It is not created on a per-entity basis. A *reference* to this single instance is attached to an entity by a system, typically during the entity's initial creation or spawning phase.
- **Scope:** The singleton object exists for the entire lifetime of the server application. The component's *attachment* to an entity is typically transient. It is expected to be attached, processed by a system, and then removed in a single logical operation, often spanning only a few game ticks.
- **Destruction:** The singleton instance is garbage collected when the server shuts down. The component reference is removed from an entity by a processing system after the skin has been successfully applied.

## Internal State & Concurrency
- **State:** This component is **stateless and immutable**. It contains no instance fields. Its sole purpose is to exist as a tag.
- **Thread Safety:** The class is inherently thread-safe. As a stateless singleton, multiple threads can safely access the shared `INSTANCE` without any risk of data corruption or race conditions. No locking mechanisms are required.

## API Surface
The public contract is defined by its static members and its implementation of the Component interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| INSTANCE | ApplyRandomSkinPersistedComponent | O(1) | The canonical singleton instance of the component. |
| CODEC | BuilderCodec | O(1) | The serializer/deserializer used by the persistence layer. Always resolves to the singleton INSTANCE. |
| getComponentType() | ComponentType | O(1) | Retrieves the registered type identifier for this component from the core EntityModule. |
| clone() | Component | O(1) | Returns the singleton `INSTANCE`. This is a no-op clone required by the Component interface. |

## Integration Patterns

### Standard Usage
This component is not meant to be interacted with directly. It is attached to an entity to trigger a system. A system responsible for player appearance would query for entities possessing this component.

```java
// Inside a hypothetical PlayerSkinSystem
// This code is illustrative of the concept

for (Entity entity : world.query(ApplyRandomSkinPersistedComponent.class)) {
    // 1. Get or create the component that holds the actual skin data
    SkinComponent skin = entity.getOrCreate(SkinComponent.class);

    // 2. Apply the business logic
    skin.setSkinId(skinService.getRandomSkin());

    // 3. CRITICAL: Remove the tag component to prevent re-processing
    entity.remove(ApplyRandomSkinPersistedComponent.class);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ApplyRandomSkinPersistedComponent()`. This defeats the purpose of the singleton pattern and creates unnecessary garbage. Always use `ApplyRandomSkinPersistedComponent.INSTANCE`.
- **Adding State:** Do not modify this class to add fields. It is designed as a stateless tag. If you need to store the result of the skin selection, use a separate data component (e.g., a SkinComponent).
- **Leaving Component Attached:** Failure to remove this component after processing will cause the skin randomization logic to run repeatedly every time the responsible system executes its query, leading to unintended behavior and performance degradation.

## Data Pipeline
This component acts as a trigger within a larger data flow for entity initialization. It does not transform data itself.

> Flow:
> Entity Creation Event -> PlayerFactorySystem attaches **ApplyRandomSkinPersistedComponent** -> Entity is serialized by EntityStore -> On next tick, PlayerSkinSystem queries for entities with this component -> System applies a random skin and **removes** the component -> The entity's final state (with the new skin and without this tag) is persisted.

