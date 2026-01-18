---
description: Architectural reference for the Invulnerable component.
---

# Invulnerable

**Package:** com.hypixel.hytale.server.core.modules.entity.component
**Type:** Singleton

## Definition
```java
// Signature
public class Invulnerable implements Component<EntityStore> {
```

## Architecture & Concepts
The Invulnerable class is a server-side **Tag Component** within Hytale's Entity Component System (ECS). Unlike data components that store state (e.g., Health, Position), a tag component's sole purpose is to mark an entity with a specific, boolean characteristic. Its presence on an entity signifies that the entity cannot take damage; its absence signifies normal damage rules apply.

Architecturally, this component is designed as an immutable singleton. This is a significant performance and memory optimization. Since the component itself holds no unique data per entity, a single shared instance, Invulnerable.INSTANCE, can be attached to any number of entities without incurring the overhead of repeated object allocation.

Systems responsible for damage calculation, such as combat or environmental effect processors, will query an entity for the presence of the Invulnerable component type. If the component is found, the damage logic is bypassed for that entity.

## Lifecycle & Ownership
- **Creation:** The singleton Invulnerable.INSTANCE is instantiated once by the Java ClassLoader when the Invulnerable class is first loaded into memory. It is not created on a per-entity basis.
- **Scope:** The single instance persists for the entire lifetime of the server process.
- **Destruction:** The object is eligible for garbage collection only when the server is shutting down and its ClassLoader is unloaded.

## Internal State & Concurrency
- **State:** This component is **stateless and immutable**. It contains no fields and its identity is defined entirely by its type. This design is fundamental to its role as a reusable tag.
- **Thread Safety:** The class is inherently thread-safe. As an immutable singleton, it can be safely referenced and checked by any system on any thread without locks or synchronization primitives.

## API Surface
The public contract is minimal, reinforcing its role as a passive data tag.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | ComponentType | O(1) | Static accessor to retrieve the globally registered type for this component from the EntityModule. This is the primary symbol used to interact with the component system. |
| clone() | Component | O(1) | Returns the singleton INSTANCE. This override ensures that attempts to clone the component for an entity do not create new objects, preserving the singleton pattern. |

## Integration Patterns

### Standard Usage
Developers should never interact with the Invulnerable instance directly. Instead, they interact with the entity's component container using the component's registered type.

```java
// Correct: Adding or removing the component from an entity
// Note: The exact API may vary, this is a conceptual example.

Entity targetEntity = ...;
ComponentType invulnerableType = Invulnerable.getComponentType();

// To make an entity invulnerable
targetEntity.addComponent(invulnerableType);

// To check if an entity is invulnerable
boolean isInvulnerable = targetEntity.hasComponent(invulnerableType);

// To remove invulnerability
targetEntity.removeComponent(invulnerableType);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create an instance. The constructor is private to enforce the singleton pattern. `new Invulnerable()` will result in a compile-time error.
- **State Assumption:** Do not attempt to cast this component and look for state. It has none. Its presence is the only information it conveys.
- **Redundant Checks:** Avoid checking `entity.getComponent(Invulnerable.getComponentType()) != null`. The more idiomatic and potentially optimized method is `entity.hasComponent(Invulnerable.getComponentType())`.

## Data Pipeline
The Invulnerable component does not process data; it is a piece of metadata used by other systems to alter their data processing pipelines.

> **Damage Calculation Flow:**
> Damage Source -> Combat System -> Target Entity Query -> **Check for Invulnerable.class** -> (If Present) -> Abort Damage Calculation -> (If Absent) -> Proceed with Damage Calculation

> **Persistence Flow:**
> Server Shutdown/Chunk Unload -> Entity Serialization -> **Invulnerable component detected** -> `CODEC` writes component type ID to `EntityStore` -> Entity Saved

