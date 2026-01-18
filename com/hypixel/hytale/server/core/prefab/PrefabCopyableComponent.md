---
description: Architectural reference for PrefabCopyableComponent
---

# PrefabCopyableComponent

**Package:** com.hypixel.hytale.server.core.prefab
**Type:** Singleton

## Definition
```java
// Signature
public class PrefabCopyableComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The PrefabCopyableComponent is a stateless **marker component** used by the server-side entity system. It does not hold any data itself; its sole purpose is to act as a tag or flag on other components within a prefab definition.

When the server instantiates a new entity from a prefab, it iterates through the components defined in that prefab. The presence of the PrefabCopyableComponent on a source component signals to the entity instantiation logic that the source component should be deep-copied to the new entity. Components within a prefab that *lack* this marker are typically ignored or handled by separate initialization logic, preventing the duplication of transient or runtime-specific state.

This class is a fundamental building block of the prefab system, providing a declarative mechanism for controlling entity composition at spawn time. It decouples the definition of a prefab from the runtime logic that creates entities, allowing designers to specify duplication behavior directly in the asset data.

## Lifecycle & Ownership
- **Creation:** A single, static instance is eagerly instantiated by the Java Virtual Machine when the PrefabCopyableComponent class is loaded. This occurs once during server bootstrap.

- **Scope:** Application-scoped. The singleton instance, referenced by the static final field INSTANCE, persists for the entire lifetime of the server process.

- **Destruction:** The component is never explicitly destroyed. It is eligible for garbage collection only when the server shuts down and its class loader is unloaded.

## Internal State & Concurrency
- **State:** This component is **stateless and immutable**. It contains no instance fields and its behavior is constant. All entities that possess this component share the exact same global instance.

- **Thread Safety:** This class is inherently thread-safe. As a stateless singleton, it can be safely accessed and referenced from any thread without locks or synchronization primitives.

## API Surface
The primary interaction with this component is through its static methods or by checking for its presence on other components.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally registered ComponentType for this component from the EntityModule. |
| get() | static PrefabCopyableComponent | O(1) | Returns the global singleton instance of this component. |
| clone() | PrefabCopyableComponent | O(1) | **Warning:** This method does not create a new object. It returns the shared singleton instance, enforcing the single-instance guarantee. |

## Integration Patterns

### Standard Usage
This component is not intended for direct manipulation by gameplay systems. Instead, it is queried by low-level entity creation services. The following example illustrates the conceptual check performed by the engine.

```java
// Hypothetical engine code for instantiating an entity from a prefab
void spawnFromPrefab(Prefab prefab, Entity newEntity) {
    for (Component<?> sourceComponent : prefab.getComponents()) {
        // Check if the component is marked as copyable
        if (sourceComponent.hasComponent(PrefabCopyableComponent.getComponentType())) {
            // If so, clone it and add it to the new entity
            newEntity.addComponent(sourceComponent.clone());
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new PrefabCopyableComponent()`. This creates a redundant, unmanaged object and bypasses the singleton pattern. Always use the static `PrefabCopyableComponent.get()` method.

- **Adding State:** Do not subclass this component to add fields or state. The entity system relies on its stateless, marker-like nature. Introducing state would violate its core design contract.

- **Runtime Manipulation:** Avoid adding or removing this component from entities during gameplay. Its presence should be treated as a static property defined in the prefab asset data.

## Data Pipeline
This component does not process a data stream. Instead, it acts as a conditional gate within the entity creation data flow, directing which components from a prefab are duplicated.

> Flow:
> Prefab Asset (Data File) -> Prefab Loading Service -> Entity Instantiation Request -> **Query for PrefabCopyableComponent** -> Conditional Component Cloning -> Newly Created Entity

