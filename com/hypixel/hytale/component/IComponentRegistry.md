---
description: Architectural reference for IComponentRegistry
---

# IComponentRegistry

**Package:** com.hypixel.hytale.component
**Type:** Core Service Interface

## Definition
```java
// Signature
public interface IComponentRegistry<ECS_TYPE> {
    // Methods omitted for brevity
}
```

## Architecture & Concepts

The IComponentRegistry interface is the foundational contract for the Entity Component System (ECS). It serves as the central, authoritative schema definition service for a given ECS world instance. Its primary responsibility is to assign unique, stable identifiers and metadata to every type of Component, Resource, System, and Event that can exist within that world.

This interface does not manage the *instances* of components, but rather the *definitions* or *types*. It acts as a type manifest, ensuring that all parts of the engine—from serialization and networking to game logic—agree on what a specific component is and how to handle it.

The generic parameter, ECS_TYPE, is a context marker. It scopes a registry implementation to a specific environment, such as a client world or a server world. This powerful design prevents type definition conflicts and allows for client-only or server-only ECS elements to be registered without ambiguity. An implementation of this interface is the first critical service to be configured during the bootstrap of any ECS world.

## Lifecycle & Ownership

- **Creation:** An implementation of IComponentRegistry is instantiated by the bootstrap logic responsible for creating an ECS world (e.g., ClientWorldFactory or ServerWorldFactory). It is one of the first objects created for a new world.
- **Scope:** The registry's lifetime is tightly bound to its corresponding ECS world. It persists for the entire duration of the world's existence.
- **Destruction:** The registry is eligible for garbage collection only when the ECS world it belongs to is fully unloaded and destroyed. There is no explicit public destruction method; its lifecycle is managed entirely by the parent world container.

## Internal State & Concurrency

- **State:** As an interface, IComponentRegistry is stateless. However, any concrete implementation is inherently stateful. It maintains internal maps or lookup tables that associate class definitions with their generated type identifiers, codecs, and factory functions. This state is built during a specific initialization phase and is considered immutable thereafter.
- **Thread Safety:** Implementations are not expected to be thread-safe during the initial registration phase. All registration calls **must** occur from a single thread during the engine's bootstrap sequence. After this phase, the registry's internal state is effectively frozen, and all read operations (e.g., looking up a type by its class) **must** be thread-safe for concurrent access by multiple game systems.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerComponent(class, supplier) | ComponentType | O(1) | Registers a simple component type with a default constructor. Used for components that do not require serialization. |
| registerComponent(class, name, codec) | ComponentType | O(1) | Registers a component type with a unique string name and a BuilderCodec for serialization and networking. **This is the required method for any component that persists or is sent over the network.** |
| registerResource(class, supplier) | ResourceType | O(1) | Registers a world-global resource with a default constructor. |
| registerResource(class, name, codec) | ResourceType | O(1) | Registers a serializable, world-global resource. |
| registerSystemType(class) | SystemType | O(1) | Registers a system's class, allowing it to be identified and managed by the ECS scheduler. |
| registerEntityEventType(class) | EntityEventType | O(1) | Registers an event type that can be dispatched to a specific entity. |
| registerWorldEventType(class) | WorldEventType | O(1) | Registers an event type that can be dispatched globally to the world. |
| registerSpatialResource(...) | ResourceType | O(1) | A specialized method to register the engine's core spatial partitioning resource. This is an engine-internal API. |

## Integration Patterns

### Standard Usage

The registry is used during the initialization phase of the game or a mod to define all custom ECS elements. The correct pattern is to retrieve the world-specific registry and perform all registrations within a dedicated setup block.

```java
// In a client-side initialization routine
IComponentRegistry<ClientECS> clientRegistry = clientWorld.getComponentRegistry();

// Register a component that will be networked
clientRegistry.registerComponent(
    PlayerInputComponent.class,
    "hytale:player_input",
    PlayerInputComponent.CODEC
);

// Register a client-only system
clientRegistry.registerSystemType(RenderingSystem.class);
```

### Anti-Patterns (Do NOT do this)

- **Late Registration:** Do not call any registration methods after the ECS world has started its update loop. This will lead to undefined behavior, as running systems will not be aware of the new type. All registrations must occur during the designated bootstrap phase.
- **Direct Implementation:** Do not attempt to implement this interface yourself unless you are building a custom game engine. Always use the provided implementation from the world object.
- **Omitting Codecs:** Do not use the Supplier-based registration for components that need to be saved to disk or synchronized between the client and server. The system will fail to serialize them, causing crashes or data loss.

## Data Pipeline

The IComponentRegistry is not a step in a data processing pipeline; rather, it is the **source of metadata** that enables those pipelines. It provides the foundational type information required by other critical systems.

> Flow:
> **IComponentRegistry** -> Type Definitions -> Used by:
> - **Serializer:** To know how to encode/decode components for disk storage.
> - **Network Subsystem:** To know how to serialize/deserialize components for network replication.
> - **ECS World:** To know how to construct and manage component storage arrays.
> - **Scripting Engine:** To expose component types to the scripting environment.

