---
description: Architectural reference for UniqueItemUsagesComponent
---

# UniqueItemUsagesComponent

**Package:** com.hypixel.hytale.server.core.entity.entities.player.data
**Type:** Data Component

## Definition
```java
// Signature
public class UniqueItemUsagesComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The UniqueItemUsagesComponent is a server-side data component within Hytale's Entity Component System (ECS). Its sole responsibility is to track the set of "unique" items a player entity has used. This mechanism is fundamental for systems that rely on one-time actions or persistent player memory, such as achievements, quest progression, or unlocking recipes.

As a pure data container, it does not implement any game logic itself. Instead, other game systems (e.g., a QuestSystem or an AchievementSystem) query this component to make stateful decisions based on the player's history.

The presence of a static CODEC field indicates that this component's state is designed for persistence. The server serializes its data into the EntityStore when a player is unloaded and deserializes it upon loading, ensuring that the player's history of used unique items is maintained across sessions.

### Lifecycle & Ownership
- **Creation:** An instance of UniqueItemUsagesComponent is created under two circumstances:
    1.  **Deserialization:** The primary creation path. When a player entity is loaded from the EntityStore, the component's CODEC is invoked to construct a new instance from saved data.
    2.  **Initial Attachment:** When a new player entity is created for the first time, the ECS framework attaches a default, empty instance of this component.
- **Scope:** The component's lifetime is strictly bound to the player entity it is attached to. It exists in memory only as long as the player entity is active on the server.
- **Destruction:** The Java object is eligible for garbage collection when the parent player entity is unloaded from the server. Prior to this, its internal state is serialized via its CODEC and persisted to the EntityStore.

## Internal State & Concurrency
- **State:** The component's state is mutable. It consists of a single internal field, a HashSet named usedUniqueItems, which stores the string identifiers of items that have been used. The state grows as the player uses more unique items.
- **Thread Safety:** This component is **not thread-safe**. The internal HashSet is not a concurrent collection. All interactions, including reads via hasUsedUniqueItem and writes via recordUniqueItemUsage, must be performed on the entity's owning server thread (the main game tick thread).

**WARNING:** Accessing this component from asynchronous tasks or other threads without external synchronization will lead to race conditions, potential ConcurrentModificationExceptions, and permanent data corruption. The engine's ECS framework is expected to enforce this single-threaded access model.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally unique type identifier for this component from the EntityModule registry. |
| clone() | Component | O(N) | Creates a deep copy of the component, where N is the number of items used. Used internally by the engine for entity duplication. |
| hasUsedUniqueItem(String) | boolean | O(1) | Checks if a given item ID has been previously recorded. This is an average-case complexity for a HashSet lookup. |
| recordUniqueItemUsage(String) | void | O(1) | Adds a new item ID to the set of used items. This is an average-case complexity for a HashSet insertion. |

## Integration Patterns

### Standard Usage
Game logic should always retrieve the component from an entity instance. Never store a long-lived reference to a component, as the parent entity may be unloaded.

```java
// Example: A quest system checking if a player has used a key
Player playerEntity = ...;
UniqueItemUsagesComponent usageData = playerEntity.getComponent(UniqueItemUsagesComponent.getComponentType());

if (usageData != null && !usageData.hasUsedUniqueItem("ancient_ruins_key")) {
    // Logic to consume the key and open the door
    usageData.recordUniqueItemUsage("ancient_ruins_key");
    playerEntity.saveComponent(usageData); // Mark component as dirty for persistence
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new UniqueItemUsagesComponent()`. The ECS framework and the persistence layer are responsible for the component's lifecycle. Manually creating an instance will result in an untracked component that is not persisted.
- **Cross-Thread Modification:** Do not modify the component from a network thread, a separate worker thread, or an asynchronous callback. All mutations must be scheduled to run on the main server tick.

## Data Pipeline
The component's data flows between in-memory representation and persistent storage.

> **Serialization (Player Save):**
> Game Logic calls `recordUniqueItemUsage()` -> In-memory `HashSet` is modified -> Server triggers entity unload -> **UniqueItemUsagesComponent.CODEC** is invoked -> `HashSet` is converted to a `String[]` -> Serialized data is written to the `EntityStore`.

> **Deserialization (Player Load):**
> Server triggers entity load -> Data is read from `EntityStore` -> **UniqueItemUsagesComponent.CODEC** is invoked -> A new `UniqueItemUsagesComponent` is instantiated -> The `String[]` from storage is used to populate the internal `HashSet` -> The fully-formed component is attached to the player entity.

