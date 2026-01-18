---
description: Architectural reference for CreativeHubEntityConfig
---

# CreativeHubEntityConfig

**Package:** com.hypixel.hytale.builtin.creativehub.config
**Type:** Data Component

## Definition
```java
// Signature
public class CreativeHubEntityConfig implements Component<EntityStore> {
```

## Architecture & Concepts
The CreativeHubEntityConfig is a data-only component within Hytale's Entity Component System (ECS). Its sole purpose is to attach persistent configuration data to an entity that represents a world, specifically an **EntityStore**.

This component acts as a data tag, creating a link between a specific world (such as a player's creative plot) and its parent "hub" world. This relationship is stored in the `parentHubWorldUuid` field. It does not contain any logic; it is a passive data container manipulated by higher-level game systems within the CreativeHub plugin.

Serialization for persistence is handled declaratively through the static **CODEC** field. The engine's world-saving mechanism automatically discovers this component on an **EntityStore**, invokes its codec, and writes the data to the world's storage. This decouples the data definition from the persistence logic.

## Lifecycle & Ownership
- **Creation:** An instance of CreativeHubEntityConfig is never created directly using its constructor. It is lazily instantiated by the ECS framework via the static `ensureAndGet` method. This method atomically gets an existing component or creates and attaches a new one to the target **Holder** (the entity).

- **Scope:** The component's lifecycle is strictly bound to the **EntityStore** entity it is attached to. It persists as long as the entity exists in the game world.

- **Destruction:** The component is marked for garbage collection when its parent entity is destroyed or if the component is explicitly removed from the entity by a game system.

## Internal State & Concurrency
- **State:** The component holds a single piece of mutable state: the `parentHubWorldUuid`. This field is nullable, indicating that a world may not have a parent hub. The state is intended to be written once during world creation and read thereafter, though it can be modified at any time.

- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking mechanisms. All interactions with components, including this one, must be performed on the primary world thread to prevent race conditions and data corruption. The Hytale engine's ECS architecture enforces this single-threaded access model for entity state modification.

## API Surface
The primary contract is defined by the static accessor methods, which are the designated entry points for interacting with this component.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ensureAndGet(Holder) | CreativeHubEntityConfig | O(1) | **Primary Accessor.** Retrieves or creates the component on an entity. This is the standard method for obtaining a handle to the component. |
| get(Holder) | CreativeHubEntityConfig | O(1) | Retrieves the component from an entity if it exists, otherwise returns null. Use this for read-only checks. |
| getParentHubWorldUuid() | UUID | O(1) | Returns the configured UUID of the parent hub world. May be null. |
| setParentHubWorldUuid(UUID) | void | O(1) | Sets the UUID of the parent hub world. This immediately mutates the component's state. |

## Integration Patterns

### Standard Usage
To associate a world entity with a parent hub, retrieve the component from the entity's **Holder** and set the property. The `ensureAndGet` method guarantees a non-null component instance.

```java
// 'worldEntityHolder' is a Holder<EntityStore> for the target world
UUID hubId = ...; // The UUID of the parent hub

// Get or create the component on the world entity
CreativeHubEntityConfig config = CreativeHubEntityConfig.ensureAndGet(worldEntityHolder);

// Set the configuration data
config.setParentHubWorldUuid(hubId);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new CreativeHubEntityConfig()`. This creates an orphaned object that is not managed by the ECS, will not be persisted, and will be garbage collected unpredictably.
- **State Caching:** Do not retrieve a component and store a reference to it in a long-lived service. Components can be removed from entities at any time. Always re-fetch the component from the entity **Holder** when you need to access it.
- **Cross-Thread Modification:** Do not access or modify this component from any thread other than the main world thread. Doing so will lead to severe concurrency bugs.

## Data Pipeline
This component's primary role is in the data persistence pipeline for world configuration. It serves as the in-memory representation of data that is serialized to disk.

> Flow:
> Game Logic (World Creation) -> `ensureAndGet` -> `setParentHubWorldUuid` -> **CreativeHubEntityConfig** (In-Memory State) -> World Save Event -> ECS Serializer -> **CODEC** -> Persistent World Storage (Disk)

