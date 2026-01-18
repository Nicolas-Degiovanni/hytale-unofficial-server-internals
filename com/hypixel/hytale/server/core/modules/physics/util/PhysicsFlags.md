---
description: Architectural reference for PhysicsFlags
---

# PhysicsFlags

**Package:** com.hypixel.hytale.server.core.modules.physics.util
**Type:** Utility

## Definition
```java
// Signature
public class PhysicsFlags {
```

## Architecture & Concepts
The PhysicsFlags class serves as a centralized, authoritative definition for collision masks within the server-side physics engine. It is a non-instantiable utility class that provides a set of compile-time constants representing distinct collision behaviors.

Its primary architectural function is to decouple game logic systems from the low-level implementation details of the physics module. Instead of scattering magic numbers (e.g., 0, 1, 2, 3) throughout the codebase to define collision properties, systems reference these static, human-readable flags. This improves code clarity, reduces the risk of errors, and simplifies future modifications to the collision system.

The flags are designed to be used as bitmasks:
- **ENTITY_COLLISIONS** represents bit 0 (value 1).
- **BLOCK_COLLISIONS** represents bit 1 (value 2).
- **ALL_COLLISIONS** is a combination of the entity and block bits (value 3).

This bitwise design allows the physics engine to perform efficient checks to determine if an object should collide with entities, blocks, both, or neither.

### Lifecycle & Ownership
- **Creation:** As a class containing only static final members, PhysicsFlags is never instantiated. Its constants are loaded into memory by the JVM ClassLoader when the class is first referenced by any part of the server application.
- **Scope:** The flags are application-scoped and globally accessible. They persist for the entire lifetime of the server process.
- **Destruction:** The class definition and its associated static data are unloaded from memory when the application's ClassLoader is garbage collected, which typically occurs only at server shutdown.

## Internal State & Concurrency
- **State:** The class has no instance state. Its static state consists of immutable, compile-time integer constants. It is therefore considered **Immutable**.
- **Thread Safety:** PhysicsFlags is inherently **Thread-Safe**. All members are `static final` primitives. Accessing these constants from any thread is a safe, atomic read operation that requires no synchronization or locking mechanisms.

## API Surface
The public contract consists exclusively of static constant fields.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| NO_COLLISIONS | static final int | O(1) | A mask that disables all physics-based collisions for an object. |
| ENTITY_COLLISIONS | static final int | O(1) | A mask that enables collisions only with other entities. |
| BLOCK_COLLISIONS | static final int | O(1) | A mask that enables collisions only with static world geometry (blocks). |
| ALL_COLLISIONS | static final int | O(1) | A mask that enables all available collision types. |

## Integration Patterns

### Standard Usage
These flags are intended to be used when configuring the physical properties of an entity or when performing collision queries. The consumer of the flag does not need to understand the underlying bitwise values.

```java
// Example: Configuring an entity to only collide with blocks
EntityPhysicsComponent physics = entity.getComponent(EntityPhysicsComponent.class);
physics.setCollisionMask(PhysicsFlags.BLOCK_COLLISIONS);

// Example: Checking if a mask includes entity collisions
int currentMask = physics.getCollisionMask();
if ((currentMask & PhysicsFlags.ENTITY_COLLISIONS) != 0) {
    // Logic for entity-aware objects
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never attempt to create an instance with `new PhysicsFlags()`. This class is not designed for instantiation and provides no instance-level functionality.
- **Using Magic Numbers:** Avoid using raw integer literals where a flag is expected. This defeats the purpose of the class and introduces brittleness.
  - **BAD:** `physics.setCollisionMask(2);`
  - **GOOD:** `physics.setCollisionMask(PhysicsFlags.BLOCK_COLLISIONS);`

## Data Pipeline
PhysicsFlags is not a processor in a data pipeline; rather, it is a static source of configuration data that influences the behavior of the pipeline.

> Flow:
> **PhysicsFlags** -> Entity Configuration -> Physics Engine -> Collision Resolution Logic

