---
description: Architectural reference for PrefabAnchor
---

# PrefabAnchor

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor
**Type:** Singleton

## Definition
```java
// Signature
public class PrefabAnchor implements Component<EntityStore> {
```

## Architecture & Concepts
The PrefabAnchor is a **marker component** within Hytale's Entity Component System (ECS). It contains no data fields; its sole purpose is to be attached to an entity to flag it as the positional origin, or "anchor", of a prefab structure. This pattern is used extensively by the server-side Builder Tools.

Architecturally, it serves as a simple, low-cost signaling mechanism. Systems responsible for saving, loading, or manipulating prefabs can efficiently query the world for the single entity that possesses this component. The transform (position, rotation) of that entity then becomes the reference point for all other blocks and entities within the prefab, defining its local coordinate space.

Its implementation as a stateless singleton is a critical performance optimization. Since the component itself holds no unique data per entity, a single shared instance can be attached to any entity without incurring memory allocation overhead. The `CODEC` and `clone` implementations are specifically designed to enforce and leverage this singleton pattern during serialization and entity duplication.

### Lifecycle & Ownership
- **Creation:** The static `INSTANCE` is created once by the JVM during class initialization. As a component, it is not instantiated per-entity. Instead, a *reference* to the singleton `INSTANCE` is attached to an entity by a builder tool system, typically in response to a user command within the prefab editor.
- **Scope:** The singleton `INSTANCE` is application-scoped and persists as long as the `BuilderToolsPlugin` is loaded. The component's attachment to an entity is scoped to the lifecycle of that entity.
- **Destruction:** The singleton instance is garbage collected only when its ClassLoader is unloaded. The component is *detached* from an entity when the entity is destroyed or when a user action explicitly removes the anchor point.

## Internal State & Concurrency
- **State:** Immutable and stateless. The PrefabAnchor class contains no instance-level fields. This is fundamental to its design as a shared, singleton component.
- **Thread Safety:** Inherently thread-safe. As a stateless singleton, the `INSTANCE` can be safely accessed and referenced from any thread without synchronization primitives. Concurrency concerns are managed by the underlying ECS framework (the `EntityStore`) which handles the safe attachment and detachment of components.

## API Surface
The primary interaction with this class is not through direct method calls on its instance, but through the ECS type system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the registered type definition for this component. This is the canonical way to reference the component type when interacting with the ECS. |

## Integration Patterns

### Standard Usage
Systems should never hold a direct reference to the `PrefabAnchor.INSTANCE`. Instead, they should query the `EntityStore` for entities that possess the component type.

```java
// A system finds the prefab's origin entity to calculate relative positions.
ComponentType<EntityStore, PrefabAnchor> anchorType = PrefabAnchor.getComponentType();

// The query returns a view of all entities with the PrefabAnchor component.
// In a valid prefab, this should only ever be one entity.
for (Entity entity : entityStore.view(anchorType)) {
    Transform transform = entity.getTransform();
    // ... use transform as the prefab's origin (0,0,0)
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Do not use reflection or other means to create new instances. This violates the core singleton design and will cause unpredictable behavior in systems that rely on reference equality (`== INSTANCE`).
- **Adding State:** Do not subclass PrefabAnchor to add state. Its stateless nature is intentional. If you need to store data related to a prefab's anchor, create a separate data component (e.g., `PrefabAnchorMetadata`) and attach it to the same entity.

## Data Pipeline
The PrefabAnchor does not process data itself. Instead, its *presence* on an entity is the data, signaling a key piece of information in a larger data flow.

> Flow:
> User Action (e.g., "Set Anchor" in UI) -> Network Command -> Server-side `PrefabEditorSystem` -> **Attaches `PrefabAnchor` component to target Entity** -> `PrefabExportSystem` queries for entity with `PrefabAnchor` -> Entity's Transform is used as origin for prefab serialization -> Serialized Prefab Data

