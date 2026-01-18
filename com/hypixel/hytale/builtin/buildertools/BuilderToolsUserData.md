---
description: Architectural reference for BuilderToolsUserData
---

# BuilderToolsUserData

**Package:** com.hypixel.hytale.builtin.buildertools
**Type:** Component (Data Object)

## Definition
```java
// Signature
public class BuilderToolsUserData implements Component<EntityStore> {
```

## Architecture & Concepts
BuilderToolsUserData is a data-holding **Component** within Hytale's Entity-Component-System (ECS) architecture. Its sole purpose is to store per-player configuration settings for the in-game Builder Tools feature. It does not contain any logic; it is a Plain Old Java Object (POJO) that represents a player's preferences.

This component is designed to be attached to a **Player** entity. The engine's persistence layer, represented by the generic type parameter EntityStore, is responsible for saving and loading this component's state as part of the player's data.

The static **CODEC** field is a critical element, defining how this object is serialized to and deserialized from a persistent format. This allows player settings to be saved to the world database and restored across sessions. The component's lifecycle and management are intrinsically linked to the BuilderToolsPlugin, which registers its type with the engine's component system.

## Lifecycle & Ownership
- **Creation:** An instance of BuilderToolsUserData is created and attached to a Player entity by the server's component management system. The static `get(Player)` method provides a safe way to access the component, returning a default-constructed, unattached instance if one does not already exist on the player. The engine is responsible for attaching and managing the canonical instance.
- **Scope:** The component's lifetime is bound to the Player entity to which it is attached. It persists as long as the player's data is loaded in the world.
- **Destruction:** The component is destroyed and marked for garbage collection when the parent Player entity is unloaded or removed from the world.

## Internal State & Concurrency
- **State:** The internal state is **mutable**. It consists of a single boolean flag, `selectionHistory`, which can be changed at runtime. It holds no caches or complex data structures.
- **Thread Safety:** This class is **not thread-safe**. As a standard ECS component, it is designed to be accessed and modified exclusively by the main server thread that owns the corresponding Player entity. Unsynchronized access from other threads will lead to data corruption and unpredictable behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(Player player) | static BuilderToolsUserData | O(log N) | Safely retrieves the component from a player. Returns a new default instance if not found. |
| getComponentType() | static ComponentType | O(1) | Retrieves the registered type identifier for this component. |
| isRecordingSelectionHistory() | boolean | O(1) | Returns the current value of the selection history setting. |
| setRecordSelectionHistory(boolean) | void | O(1) | Modifies the selection history setting. |
| clone() | Component | O(1) | Creates a deep copy of this component's data. |

## Integration Patterns

### Standard Usage
The correct pattern for interacting with this component is to retrieve it from an existing Player entity. This ensures you are working with the player's actual, managed data, not a transient copy.

```java
// How a developer should normally use this
Player targetPlayer = ...;
BuilderToolsUserData settings = BuilderToolsUserData.get(targetPlayer);

boolean isRecording = settings.isRecordingSelectionHistory();

// Modify the player's setting
settings.setRecordSelectionHistory(false);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BuilderToolsUserData()` in game logic. This creates an orphaned object that is not attached to any player and whose changes will not be persisted. Always use the static `get(Player)` method.
- **State Caching:** Do not retrieve the component once and store a reference to it for a long duration. The component could be removed or replaced. Always re-fetch it from the Player entity when you need to access it.

## Data Pipeline
BuilderToolsUserData primarily functions as a data payload within the server's persistence and entity management pipeline. Its state is serialized on world save and deserialized on player load.

> Flow:
> Player Data Load -> EntityStore reads from disk -> **CODEC** deserializes data -> **BuilderToolsUserData** instance attached to Player -> Game logic reads settings -> Player action modifies settings -> Component state is updated -> World Save Event -> **CODEC** serializes **BuilderToolsUserData** -> EntityStore writes to disk.

