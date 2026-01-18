---
description: Architectural reference for FromPrefab
---

# FromPrefab

**Package:** com.hypixel.hytale.server.core.modules.entity.component
**Type:** Singleton

## Definition
```java
// Signature
public class FromPrefab implements Component<EntityStore> {
```

## Architecture & Concepts
The FromPrefab component is a server-side **marker component** within the Entity-Component-System (ECS) framework. Its sole purpose is to tag an entity, signifying that its initial state was derived from a prefab asset file. It contains no data; its presence on an entity is the only information it conveys.

Architecturally, this class embodies the Flyweight pattern through its implementation as a stateless singleton. Because all instances are identical and carry no state, a single shared instance, FromPrefab.INSTANCE, is used for every entity that requires this tag. This design significantly reduces memory overhead, as thousands of entities can share this component without allocating new objects.

It is registered with and managed by the EntityModule, which provides a centralized ComponentType for efficient querying and type identification within the game world.

### Lifecycle & Ownership
- **Creation:** The singleton INSTANCE is created once during static class initialization when the FromPrefab class is first loaded by the Java Virtual Machine. It is not created on a per-entity basis.
- **Scope:** The component's lifecycle is tied to the server process itself. It persists for the entire duration of the server session.
- **Destruction:** The singleton instance is garbage collected only when the server's class loader is unloaded, typically during a full server shutdown.

## Internal State & Concurrency
- **State:** This component is **immutable and stateless**. It has no instance fields and its behavior never changes. The clone method returns the static INSTANCE, reinforcing its stateless nature.
- **Thread Safety:** FromPrefab is inherently thread-safe. As a stateless and immutable object, it can be safely referenced, added to, or read from entities across multiple threads without any need for locks or synchronization primitives.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally registered ComponentType for FromPrefab from the EntityModule. |
| clone() | Component | O(1) | Returns the shared singleton INSTANCE. Does not perform a deep or shallow copy. |

## Integration Patterns

### Standard Usage
This component is not meant to be instantiated. It should be added to an entity during its construction or queried by systems to alter their logic for prefab-based entities.

**Adding the component during entity creation:**
```java
// During entity spawning, add the shared instance to the entity builder.
Entity entity = entityBuilder
    .with(Transform.at(position))
    .with(FromPrefab.INSTANCE) // Attaches the singleton tag
    .build();
```

**Querying for the component in a system:**
```java
// A system can check for the presence of the component to apply specific logic.
if (entity.has(FromPrefab.getComponentType())) {
    // This entity originated from a prefab, apply special logic...
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Do not attempt to create new instances of FromPrefab via reflection or other means. This violates the singleton contract and provides no benefit.
- **State Modification:** Do not attempt to modify the component's class definition to add state. Its design as a stateless marker is fundamental to its efficiency. Adding state would break the singleton contract and introduce severe, unpredictable side effects across all entities sharing the instance.

## Data Pipeline
FromPrefab does not process a continuous stream of data. Instead, it acts as a static tag within the entity lifecycle pipeline. Its primary role is at the point of entity creation and during system queries.

> **Entity Spawning Flow:**
> Prefab Data -> Server Spawning Logic -> **FromPrefab.INSTANCE attached** -> Entity constructed -> Entity added to EntityStore

> **System Query Flow:**
> Game System Logic -> Query EntityStore (Filter by ComponentType) -> **Identify entities with FromPrefab** -> Execute prefab-specific behavior

