---
description: Architectural reference for the Emote data model.
---

# Emote

**Package:** com.hypixel.hytale.server.core.cosmetics
**Type:** Data Model

## Definition
```java
// Signature
public class Emote {
```

## Architecture & Concepts
The Emote class is a server-side, immutable data model that represents a single cosmetic emote definition. It serves as a Plain Old Java Object (POJO) whose primary responsibility is to hold the core attributes of an emote: its unique identifier, its display name, and the name of the animation asset it triggers.

Architecturally, this class is designed to be a read-only, in-memory representation of data that originates from an external data source, heavily implied to be a MongoDB database by its BSON-based constructor. It is not a service or a manager; it is the fundamental data container used by higher-level systems like an EmoteManager or CosmeticRegistry. Its immutability is a critical design choice, ensuring that emote definitions remain consistent and safe to share across the entire server application.

## Lifecycle & Ownership
- **Creation:** Emote objects are instantiated exclusively within the `com.hypixel.hytale.server.core.cosmetics` package by a factory or manager class. The constructor is `protected`, which strictly prohibits direct instantiation from outside its package. The sole creation pathway is by passing a `BsonDocument`, indicating deserialization from a database record during a data loading phase.
- **Scope:** An Emote instance typically has a global, session-wide scope. It is loaded once at server startup by a central registry and persists in memory for the entire duration of the server's runtime. It represents a *definition*, not a player-specific instance.
- **Destruction:** The object is eligible for garbage collection only when the service that loaded it (e.g., a central CosmeticRegistry) is shut down and dereferenced, which normally occurs during a full server shutdown sequence.

## Internal State & Concurrency
- **State:** The class is **effectively immutable**. Its fields (`id`, `name`, `animation`) are set only once within the constructor and there are no public or protected setters. This guarantees that an Emote object's state cannot be altered after creation.
- **Thread Safety:** The Emote class is **inherently thread-safe**. Due to its immutability, instances can be safely read, shared, and passed between multiple threads without any need for locks or other synchronization primitives. This is essential for a high-performance, multi-threaded server environment where game logic may access cosmetic data from various threads.

## API Surface
The public contract is minimal, consisting only of data accessors.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | String | O(1) | Returns the unique, non-human-readable identifier for the emote. |
| getName() | String | O(1) | Returns the human-readable, display name of the emote. |
| getAnimation() | String | O(1) | Returns the asset name of the animation to be played. |

## Integration Patterns

### Standard Usage
Developers should never instantiate an Emote directly. Instead, they must retrieve pre-existing instances from a central service or registry, typically using the emote's unique ID.

```java
// Correct: Retrieve a pre-loaded Emote definition from a manager
EmoteManager emoteManager = serverContext.getService(EmoteManager.class);
Emote waveEmote = emoteManager.getEmoteById("hytale:wave");

if (waveEmote != null) {
    // Use the immutable data
    String animationToPlay = waveEmote.getAnimation();
    player.playAnimation(animationToPlay);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The `protected` constructor makes this impossible from outside the package. Any attempt to bypass this using reflection will break the design contract and lead to an unsupported state.
- **State Modification:** Do not attempt to modify the internal fields of an Emote instance via reflection. The server's stability relies on this data being immutable. Modifying a shared Emote object would have unpredictable and catastrophic effects across all game systems.

## Data Pipeline
The Emote class is a destination point in a data loading pipeline. It transforms raw database records into strongly-typed, safe-to-use Java objects.

> Flow:
> MongoDB Record -> BSON Driver -> `BsonDocument` -> **Emote Constructor** -> In-Memory Emote Registry -> Game Logic Systems

