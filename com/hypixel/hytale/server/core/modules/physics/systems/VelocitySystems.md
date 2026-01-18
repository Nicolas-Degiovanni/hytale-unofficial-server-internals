---
description: Architectural reference for VelocitySystems.AddSystem
---

# VelocitySystems.AddSystem

**Package:** com.hypixel.hytale.server.core.modules.physics.systems
**Type:** System Component

## Definition
```java
// Signature
public static class AddSystem extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
VelocitySystems.AddSystem is a reactive system within the server-side Entity-Component-System (ECS) framework. Its sole responsibility is to enforce data integrity by ensuring that specific entities possess a Velocity component upon their creation.

This system operates on a declarative query. It targets any entity that matches the AllLegacyEntityTypesQuery but does *not* yet have a Velocity component. By subscribing to entity creation events, it acts as an automatic initializer or a data back-filling mechanism. This pattern decouples the core entity creation logic from the physics initialization logic, allowing physics behavior to be cleanly added to a wide range of entities without modifying their individual spawn routines.

It is a foundational component for the physics module, establishing the baseline state required for any subsequent physics calculations on legacy entities.

### Lifecycle & Ownership
- **Creation:** Instantiated once during the server's bootstrap sequence, typically by a SystemManager or a similar ECS orchestrator responsible for assembling the world's processing pipeline. The required ComponentType for Velocity is injected via the constructor.
- **Scope:** Session-scoped. The system remains active for the entire lifetime of the ECS world or server instance it is registered with.
- **Destruction:** The system is discarded and eligible for garbage collection when the parent ECS world is unloaded or the server shuts down. The ECS framework manages its destruction.

## Internal State & Concurrency
- **State:** The system is effectively stateless after construction. Its internal fields, velocityComponentType and query, are final and configured only at initialization. It does not cache entity data or maintain state between invocations.
- **Thread Safety:** This system is **not thread-safe** and must not be accessed from arbitrary threads. It is designed to be executed exclusively by the main server thread as part of the ECS update loop. The ECS framework guarantees that calls to its methods like onEntityAdd are serialized and occur in a controlled context, preventing race conditions within the component store.

## API Surface
The public API consists of lifecycle methods intended for invocation by the ECS framework, not by user code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, reason, store) | void | O(1) | Framework callback. Adds a default Velocity component to the entity if it matches the system's query. |
| onEntityRemoved(holder, reason, store) | void | O(1) | Framework callback. This is a no-op in this implementation. |
| getQuery() | Query | O(1) | Returns the pre-compiled query used by the framework to identify target entities for this system. |

## Integration Patterns

### Standard Usage
A developer does not interact with an instance of this system directly. Instead, it is registered with the world's SystemManager during server initialization. The framework then automatically invokes its methods when relevant events occur.

```java
// Example: During server or world setup
ComponentType<EntityStore, Velocity> velocityType = ...;
SystemManager systemManager = world.getSystemManager();

// The system is instantiated and immediately passed to the framework.
// No local reference should be retained.
systemManager.register(new VelocitySystems.AddSystem(velocityType));
```

### Anti-Patterns (Do NOT do this)
- **Manual Invocation:** Never call onEntityAdd or other lifecycle methods directly. Doing so bypasses the ECS framework's query matching and state management, which can lead to component corruption, inconsistent entity state, and unpredictable physics behavior.
- **Stateful Implementation:** Do not modify this system to store entity-specific data or other mutable state. Systems should be stateless processors that operate on the components of the entities they are given.

## Data Pipeline
This system acts as a reactive processor in the entity creation pipeline. It intercepts newly created entities that match its criteria and modifies their component state before they enter the main game loop.

> Flow:
> Entity Spawning Logic -> ECS Framework Dispatches `onEntityAdd` Event -> **AddSystem** Query Match -> `holder.ensureComponent()` -> Velocity Component Added to EntityStore

---

