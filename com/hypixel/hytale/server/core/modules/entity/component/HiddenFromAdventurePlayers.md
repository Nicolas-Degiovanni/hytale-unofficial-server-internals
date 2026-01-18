---
description: Architectural reference for HiddenFromAdventurePlayers
---

# HiddenFromAdventurePlayers

**Package:** com.hypixel.hytale.server.core.modules.entity.component
**Type:** Singleton

## Definition
```java
// Signature
public class HiddenFromAdventurePlayers implements Component<EntityStore> {
```

## Architecture & Concepts
The HiddenFromAdventurePlayers class is a **marker component** within the server-side Entity Component System (ECS). It functions as a simple, stateless tag. Its sole purpose is to flag an entity, signifying that it should be invisible and non-interactive to players in Adventure game mode.

This component contains no logic itself. Instead, higher-level systems, such as the server's visibility manager or network replication layer, query for the presence of this component on entities. Based on this check, these systems will filter the entity out of the data stream sent to relevant clients. It is a fundamental building block for controlling entity visibility based on game state.

The class is implemented as a strict singleton, ensuring that only one instance ever exists. This is highly efficient, as attaching this component to thousands of entities only involves storing a reference to the same shared object, consuming minimal memory.

### Lifecycle & Ownership
- **Creation:** The single instance is created statically during class loading by the JVM. It is not instantiated by any factory or manager at runtime.
- **Scope:** Application-wide. The `INSTANCE` field provides global access to the component, which persists for the entire server lifetime.
- **Destruction:** The object is only eligible for garbage collection when the server process shuts down and its class loader is unloaded.

## Internal State & Concurrency
- **State:** Stateless and immutable. This component has no instance fields and its state can never change. It exists only as an identifier.
- **Thread Safety:** Inherently thread-safe. As an immutable, stateless singleton, it can be safely attached, removed, or checked from any entity across any thread without locks or other synchronization primitives.

## API Surface
The public API is minimal, reflecting its role as a simple data component.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally registered ComponentType for this component from the EntityModule. This is the primary way systems identify this component type. |
| clone() | Component | O(1) | Returns the singleton INSTANCE. This enforces the singleton pattern by preventing the creation of new instances via cloning. |

## Integration Patterns

### Standard Usage
This component is not used directly. Instead, it is attached to or checked on an entity using the ECS framework APIs.

**Attaching the component to an entity:**
```java
// To make an entity invisible to adventure players, add the component's type.
// The underlying system will attach the singleton INSTANCE.
Entity myEntity = ...;
myEntity.addComponent(HiddenFromAdventurePlayers.getComponentType());
```

**Checking for the component's presence:**
```java
// A visibility system would check for the component before replicating the entity.
Entity entityToCheck = ...;
if (entityToCheck.hasComponent(HiddenFromAdventurePlayers.getComponentType())) {
    // Do not send this entity's data to adventure mode players.
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Attempting to call `new HiddenFromAdventurePlayers()` will result in a compile-time error. Always use the static `INSTANCE` or, more appropriately, the `getComponentType()` method with entity APIs.
- **Null Checks:** Do not check if `HiddenFromAdventurePlayers.INSTANCE` is null. It is a static final field initialized at class load time and is guaranteed to be non-null.
- **Stateful Extension:** Do not extend this class to add state. The server's systems rely on its identity as a stateless marker. Introducing state would violate its design contract and lead to unpredictable behavior.

## Data Pipeline
This component does not process data; it is a piece of data that influences other data pipelines. Its primary role is to act as a filter condition in the server's entity replication pipeline.

> Flow:
> Server Tick → Visibility Calculation System → For each entity, query `hasComponent(HiddenFromAdventurePlayers.getComponentType())` → **If true, entity is culled from the visibility set for adventure players** → Network Replication System → Packet sent to client without the culled entity's data.

