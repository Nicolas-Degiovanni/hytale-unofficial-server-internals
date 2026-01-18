---
description: Architectural reference for PlayerDeathPositionData
---

# PlayerDeathPositionData

**Package:** com.hypixel.hytale.server.core.entity.entities.player.data
**Type:** Data Structure / Transient

## Definition
```java
// Signature
public final class PlayerDeathPositionData {
```

## Architecture & Concepts
PlayerDeathPositionData is a final, plain data structure that serves as a serializable data contract. Its primary architectural role is to encapsulate the essential information about a player's death location: a map marker ID, the precise world transform (position and rotation), and the in-game day of the event.

This class is not a service or a manager; it is a passive data container. Its most critical feature is the static **CODEC** field, which integrates it directly into Hytale's serialization framework. This allows instances of PlayerDeathPositionData to be seamlessly encoded to and decoded from binary formats for network transmission or persistence to disk, such as in a player's save file.

By being declared **final**, the class guarantees it cannot be extended, reinforcing its purpose as a simple, predictable data record. It represents a single, immutable fact within the game world.

## Lifecycle & Ownership
- **Creation:** Instances are created under two distinct circumstances:
    1.  **By Game Logic:** When a player entity dies, the responsible game system (e.g., a death handler service) instantiates this class via its public constructor, populating it with the current world state.
    2.  **By the Serialization Framework:** When player data is loaded from a persistent source or received over the network, the Hytale `Codec` system uses the static **CODEC** definition and the private no-argument constructor to re-hydrate the object from its binary representation.

- **Scope:** This is a transient object. Its lifetime is strictly bound to its containing data structure, typically a list within a player's profile or session data. It does not persist on its own and has no global scope.

- **Destruction:** The object is managed by the Java Garbage Collector. It is marked for destruction once no more references to it exist, for example, when a player's session ends and their profile data is unloaded, or when the list of death points is cleared.

## Internal State & Concurrency
- **State:** The internal state is composed of a String, a Transform, and an integer. The class is designed to be **effectively immutable**. While the fields are not marked `final`, there are no public setters. All state is established at the moment of construction and is not intended to be modified thereafter.

- **Thread Safety:** As an effectively immutable object, an instance of PlayerDeathPositionData is safe to read from multiple threads concurrently. The data it contains, such as Transform, are also designed as value types and are safe for concurrent access. Writing is confined to the constructor, which is inherently thread-safe for the instance being created.

## API Surface
The public contract is minimal, consisting of the constructor for manual creation and getters for data retrieval. The static `CODEC` fields are the primary contract for the serialization system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PlayerDeathPositionData(markerId, transform, day) | constructor | O(1) | Creates a new instance. All arguments are non-null. |
| getMarkerId() | String | O(1) | Returns the unique identifier for the associated map marker. |
| getTransform() | Transform | O(1) | Returns the world position and orientation at the time of death. |
| getDay() | int | O(1) | Returns the in-game day number on which the death occurred. |

## Integration Patterns

### Standard Usage
This class is typically created by server-side logic in response to a player death event. The resulting instance is then added to a collection within the player's data profile, which is later persisted.

```java
// Example: A death handler system creates a new record.
Player aPlayer = ...;
World aWorld = ...;
MapMarkerService markerService = ...;

// 1. Get the state at the moment of death.
Transform deathTransform = aPlayer.getTransform();
int currentDay = aWorld.getTimeManager().getCurrentDay();
String newMarkerId = markerService.createDeathMarker(deathTransform);

// 2. Create the data object.
PlayerDeathPositionData deathData = new PlayerDeathPositionData(
    newMarkerId,
    deathTransform,
    currentDay
);

// 3. Add it to the player's persistent data.
aPlayer.getProfile().getDeathHistory().add(deathData);
```

### Anti-Patterns (Do NOT do this)
- **State Mutation:** Do not use reflection or other means to modify the internal fields of an instance after it has been created. This violates its contract as an immutable data record and can lead to unpredictable behavior in systems that rely on this data, such as save game integrity checks.
- **Manual Serialization:** Do not interact with the static **CODEC** field directly. The serialization framework is designed to discover and use it automatically. Manual invocation is verbose, error-prone, and bypasses higher-level data management APIs.

## Data Pipeline
PlayerDeathPositionData does not process data; it *is* the data payload. It exists as a structured record that flows between game logic and serialization systems.

> **Serialization Flow:**
> Player Death Event → Game Logic Creates **PlayerDeathPositionData** → Add to Player Profile → Server uses **ARRAY_CODEC** to serialize profile → Binary Data on Disk

> **Deserialization Flow:**
> Binary Data on Disk → Server uses **ARRAY_CODEC** to deserialize profile → **PlayerDeathPositionData** instances re-hydrated → Game Logic (e.g., Map UI) reads data from instances

