---
description: Architectural reference for PlayerConfigData
---

# PlayerConfigData

**Package:** com.hypixel.hytale.server.core.entity.entities.player.data
**Type:** State Container

## Definition
```java
// Signature
public final class PlayerConfigData {
```

## Architecture & Concepts

The PlayerConfigData class is a server-side state container that encapsulates all persistent data for a single player. It acts as the canonical, in-memory representation of a player's progress, settings, and discoveries. This object is not a service; it is a pure data structure designed explicitly for serialization and deserialization.

Its primary architectural feature is the static **CODEC** field, a **BuilderCodec**. This defines a robust, reflection-free contract for encoding the object's state into a binary format for storage and for decoding it back into a live object. This mechanism is central to the player persistence system.

A critical design aspect is the built-in data migration logic within the codec's **afterDecode** hook. This hook inspects the **blockIdVersion** and applies a chain of **BlockMigration** functions to upgrade older player data formats to the current version automatically upon load. This ensures forward compatibility across game updates without manual intervention.

The class also models a hierarchical state, where global player data (like known recipes) coexists with world-specific data stored in the **perWorldData** map, which contains instances of **PlayerWorldData**.

## Lifecycle & Ownership

-   **Creation:** PlayerConfigData instances are almost exclusively created by the Hytale codec system during deserialization. When a player joins a server, their saved data is read from a persistent store (e.g., a database or file) and decoded using **PlayerConfigData.CODEC**, which invokes the private constructor. Manual instantiation is a significant anti-pattern.
-   **Scope:** The object's lifetime is tied directly to a player's server session. It is loaded upon login and remains in memory, attached to the player's server-side entity, until they disconnect.
-   **Destruction:** The object is eligible for garbage collection once the player's entity is unloaded from the server. The **cleanup** method provides an explicit mechanism to prune stale data, such as references to temporary instance worlds that no longer exist, before the object is serialized for saving.

## Internal State & Concurrency

-   **State:** The internal state is highly mutable, reflecting the dynamic nature of player progress. To manage persistence efficiently, it employs a "dirty flag" pattern using the **hasChanged** AtomicBoolean. Any method that modifies a field is responsible for calling **markChanged**. A separate persistence service periodically calls **consumeHasChanged** to determine if the object's state has changed and requires saving.

-   **Thread Safety:** This class uses a mixed concurrency model and must be handled with extreme care.
    -   The dirty flag, **hasChanged**, is an AtomicBoolean, making it safe for lock-free, thread-safe checks.
    -   Certain high-traffic collections, such as **perWorldData** and **activeObjectiveUUIDs**, are implemented with concurrent collections (**ConcurrentHashMap**).
    -   However, other collections like **knownRecipes** and **discoveredZones** are standard, non-thread-safe **HashSet**s.
    -   **WARNING:** It is assumed that modifications to non-concurrent collections are performed exclusively on the main server thread. Access from other threads will lead to race conditions and data corruption.
    -   To enforce this, all public getters for collections return unmodifiable wrappers. This is a defensive pattern to prevent uncontrolled, direct mutation of the underlying collections. All modifications must go through the designated public setter methods.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPerWorldData(worldName) | PlayerWorldData | O(1) avg | Retrieves or creates world-specific player data. This is the primary entry point for accessing state tied to a specific world. |
| markChanged() | void | O(1) | Sets the internal "dirty flag", indicating the object's state has been modified and needs to be persisted. |
| consumeHasChanged() | boolean | O(1) | Atomically retrieves the current value of the dirty flag and resets it to false. This is the core method used by the persistence layer. |
| cleanup(universe) | void | O(N) | Iterates through per-world data and removes entries for worlds that no longer exist, preventing data bloat from temporary instances. |

## Integration Patterns

### Standard Usage

The PlayerConfigData object is managed by a higher-level service, such as a PlayerPersistenceManager. Game logic systems modify the player's state via its public API, and the persistence manager handles the save-on-change loop.

```java
// In a persistence service or player logout hook
public void savePlayerIfChanged(Player player) {
    PlayerConfigData data = player.getConfigData();

    // Atomically checks and resets the dirty flag
    if (data.consumeHasChanged()) {
        // Prune data for worlds that no longer exist
        data.cleanup(server.getUniverse());

        // Serialize and write to persistent storage
        byte[] serializedData = PlayerConfigData.CODEC.encode(data);
        storageBackend.save(player.getUUID(), serializedData);
    }
}

// In a game logic system (e.g., crafting)
public void onRecipeLearned(Player player, String recipeKey) {
    PlayerConfigData data = player.getConfigData();
    Set<String> newRecipes = new HashSet<>(data.getKnownRecipes());
    newRecipes.add(recipeKey);

    // Use the setter to ensure the dirty flag is marked
    data.setKnownRecipes(newRecipes);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call **new PlayerConfigData()**. This bypasses the codec's default value initialization and post-processing hooks, leading to an improperly configured object.
-   **External Modification:** Do not attempt to modify the collections returned by getter methods. They are unmodifiable and will throw an **UnsupportedOperationException**. Always create a new collection, populate it, and use the corresponding setter.
-   **Ignoring the Dirty Flag:** Modifying internal fields directly without calling **markChanged** will result in lost data, as the persistence system will not know that the object needs to be saved.
-   **Cross-Thread Writes:** Writing to non-concurrent fields (e.g., **knownRecipes**, **preset**) from any thread other than the main server thread is not safe and will cause unpredictable behavior.

## Data Pipeline

The PlayerConfigData object is the central payload in the player persistence data pipeline.

> **Load Flow:**
> Persistent Storage (File/DB) → Byte Stream → **PlayerConfigData.CODEC.decode()** → Data Migration Hook → **Live PlayerConfigData Instance** → Attached to Player Entity

> **Save Flow:**
> Game Logic calls setter → **markChanged()** → Persistence Service calls **consumeHasChanged()** → **Live PlayerConfigData Instance** → **PlayerConfigData.CODEC.encode()** → Byte Stream → Persistent Storage (File/DB)

