---
description: Architectural reference for PlayerCreativeSettings
---

# PlayerCreativeSettings

**Package:** com.hypixel.hytale.server.core.modules.entity.player
**Type:** Value Object / DTO

## Definition
```java
// Signature
public record PlayerCreativeSettings(boolean allowNPCDetection, boolean respondToHit) {
```

## Architecture & Concepts
PlayerCreativeSettings is an immutable data record that encapsulates configuration flags for a player operating in creative mode. As a Value Object, its primary role is to act as a simple, transparent carrier for a specific piece of state. It is not a service or a manager; it holds no logic beyond its own data.

Architecturally, this class is designed for safety and predictability. By being a record, it is inherently **immutable**, which prevents state corruption from concurrent modifications or unexpected side effects. Two instances of PlayerCreativeSettings with identical field values are considered equal, simplifying comparisons and state management.

This object typically exists as a component within a larger, mutable entity object, such as a Player or PlayerState. It allows systems like AI, combat, and physics to query a player's creative-mode capabilities without needing direct access to the parent entity's internal state.

## Lifecycle & Ownership
- **Creation:** A new PlayerCreativeSettings instance is created whenever a player's creative mode configuration changes. This can be triggered by server commands, game mode transitions, or when a player first joins a world with specific creative rules. The object is typically instantiated and owned by the server-side Player entity that it describes.
- **Scope:** The lifetime of a PlayerCreativeSettings object is typically short. Since it is immutable, any change results in the creation of a *new* instance, and the old one is discarded. The current instance is held by its parent Player entity and persists as long as that configuration is active.
- **Destruction:** The object is managed by the Java garbage collector. When it is no longer referenced, for example, after being replaced by a new settings object on the Player entity, it becomes eligible for collection. There are no manual cleanup requirements.

## Internal State & Concurrency
- **State:** **Immutable**. Once an instance of PlayerCreativeSettings is constructed, its state—the values of allowNPCDetection and respondToHit—cannot be altered. Any operation that appears to "modify" the settings must create and return a new instance.
- **Thread Safety:** **Inherently Thread-Safe**. Due to its immutability, a PlayerCreativeSettings object can be safely passed between and read by multiple threads without any form of external locking or synchronization. This makes it an ideal candidate for use in concurrent systems that need to read player state without risking data races.

## API Surface
The public API is minimal and consists of the canonical constructor, a default constructor, accessors, and a clone method provided by the record implementation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PlayerCreativeSettings() | constructor | O(1) | Creates a default instance with both flags set to false. |
| PlayerCreativeSettings(boolean, boolean) | constructor | O(1) | Creates an instance with the specified settings. |
| allowNPCDetection() | boolean | O(1) | Accessor for the allowNPCDetection flag. |
| respondToHit() | boolean | O(1) | Accessor for the respondToHit flag. |
| clone() | PlayerCreativeSettings | O(1) | Creates and returns a new instance with identical values. |

## Integration Patterns

### Standard Usage
This object should be treated as a transient data container. It is created to represent a new state and is typically assigned to a state holder, like a Player entity.

```java
// Example: Updating a player's settings in response to a command
Player targetPlayer = getPlayerById(playerId);

// Create a new state object; do not attempt to modify the old one
PlayerCreativeSettings newSettings = new PlayerCreativeSettings(true, false);

// Atomically replace the old settings object on the player entity
targetPlayer.setCreativeSettings(newSettings);
```

### Anti-Patterns (Do NOT do this)
- **Attempting Mutation:** The fields of a record are final. Code that attempts to modify an existing instance is a fundamental misunderstanding of the pattern and will not compile.
- **Unnecessary Cloning for Reads:** Do not clone an instance just to pass it to another system for read-only purposes. Since the object is immutable, sharing the original reference is completely safe and more performant.
- **Stateful Wrappers:** Do not wrap this object in another class that attempts to manage its state. The correct pattern is to replace the entire PlayerCreativeSettings object on its parent entity when a change is required.

## Data Pipeline
PlayerCreativeSettings is not a processing component in a pipeline; it is the data itself. It represents a snapshot of configuration that is produced by one system and consumed by others.

> Flow:
> Server Command or Game Logic -> **new PlayerCreativeSettings(...)** -> Player Entity State -> AI / Physics Systems (Read-Only Consumer)

