---
description: Architectural reference for PlayerSkin
---

# PlayerSkin

**Package:** com.hypixel.hytale.server.core.cosmetics
**Type:** Value Object / DTO

## Definition
```java
// Signature
public class PlayerSkin {
```

## Architecture & Concepts

The PlayerSkin class is an immutable data container that represents the complete cosmetic appearance of a player character. It functions as a server-side Value Object, encapsulating all the component parts that define a character's visual look, from their haircut and eyes to their clothing and accessories.

Its primary architectural role is to serve as a structured, in-memory representation of player cosmetic data that is persisted elsewhere, typically in a BSON-based document database like MongoDB. The presence of a constructor that accepts a BsonDocument is a clear indicator of its role in the data persistence layer.

The design strongly favors immutability. Once a PlayerSkin object is instantiated, its state cannot be altered. Any change to a player's appearance requires the creation of an entirely new PlayerSkin instance. This design choice guarantees thread safety and eliminates a wide class of bugs related to state management in a concurrent server environment.

The nested static class, PlayerSkinPartId, provides a standardized format for identifying individual cosmetic assets. It parses a dot-delimited string (`assetId.textureId.variantId`) into a structured object, decoupling the skin's data representation from the asset loading and rendering systems which will consume these IDs.

## Lifecycle & Ownership

-   **Creation:** A PlayerSkin object is instantiated when a player's cosmetic data is deserialized from a persistent source, such as a database lookup upon player login. It can also be created when a player modifies their character's appearance, generating a new configuration that replaces the old one.
-   **Scope:** The object's lifetime is typically tied to a player's session. An instance is held as a field within the primary player entity object. It is a transient object, not a managed service or singleton.
-   **Destruction:** The object is eligible for garbage collection as soon as it is no longer referenced. This occurs when a player's session ends or when their appearance is updated and the old PlayerSkin object is replaced by a new one.

## Internal State & Concurrency

-   **State:** **Immutable**. All internal fields are declared `final` and are assigned only once during construction. The object provides no methods for mutation.
-   **Thread Safety:** **Inherently thread-safe**. Due to its immutable nature, a PlayerSkin instance can be safely read from and shared across multiple threads without any external synchronization or locks. This is a critical feature for high-performance server systems where game logic, networking, and persistence may operate on different threads.

## API Surface

The public API consists of constructors for instantiation and getters for data retrieval. There are no mutator methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PlayerSkin(BsonDocument) | constructor | O(N) | Primary constructor for deserializing from a database record. N is the number of cosmetic parts. |
| getBodyCharacteristic() | PlayerSkinPartId | O(1) | Returns the identifier for the base body type. |
| getFace() | String | O(1) | Returns the identifier for the face asset. |
| getPants() | PlayerSkinPartId | O(1) | Returns the identifier for the pants, which may be null. |
| PlayerSkinPartId.fromString(String) | static factory | O(L) | Parses a dot-delimited string into a structured part ID. L is the length of the string. |

## Integration Patterns

### Standard Usage

The canonical use case is loading a player's data from the database, deserializing the skin information into a PlayerSkin object, and attaching it to the player's server-side entity.

```java
// Example: Loading a player's skin during login
BsonDocument playerDoc = database.findPlayer(playerId);
BsonDocument skinBson = playerDoc.getDocument("skinProfile");

// The PlayerSkin object is created once from the raw data
PlayerSkin skin = new PlayerSkin(skinBson);

// The immutable object is then attached to the live player entity
playerEntity.setSkin(skin);
```

### Anti-Patterns (Do NOT do this)

-   **Attempted Mutation:** Do not attempt to use reflection or other means to modify the fields of a PlayerSkin instance after it has been created. If a change is required, construct a new object with the updated values.
-   **Incorrect Null Handling:** Several cosmetic slots like `cape` or `gloves` can be null. Systems consuming a PlayerSkin object must be robust and handle these null values gracefully. Do not assume every `get...` method will return a non-null value.
-   **Invalid ID Propagation:** The PlayerSkin object does not validate that the asset IDs it holds correspond to actual game assets. It is merely a data container. Validation must occur in downstream systems, such as the AssetManager or the rendering pipeline.

## Data Pipeline

PlayerSkin acts as a translation point, converting raw database documents into a strongly-typed, immutable Java object for use within the server's game logic.

> Flow:
> MongoDB (BSON Document) -> Database Driver -> **PlayerSkin(BsonDocument)** -> Player Entity -> Network Serialization -> Game Client

