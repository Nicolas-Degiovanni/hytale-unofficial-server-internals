---
description: Architectural reference for PlayerSkinPartTexture
---

# PlayerSkinPartTexture

**Package:** com.hypixel.hytale.server.core.cosmetics
**Type:** Transient Data Model

## Definition
```java
// Signature
public class PlayerSkinPartTexture {
```

## Architecture & Concepts
PlayerSkinPartTexture is a simple, immutable data model that represents a single texture component of a player's cosmetic skin. Its sole responsibility is to deserialize and hold texture information from a BSON data source, which is typically a MongoDB document.

This class is a low-level building block within the server's cosmetic system. It does not contain any game logic. Instead, it serves as a structured data container, translating raw database records into a strongly-typed Java object. Higher-level systems, such as a hypothetical PlayerCosmeticProfile or SkinApplier, consume instances of this class to assemble a player's complete visual appearance. It is a classic example of a Data Transfer Object (DTO) used at the data access layer.

## Lifecycle & Ownership
- **Creation:** Instantiation is strictly controlled. The constructor is `protected`, indicating that it is only ever created by other classes within the `com.hypixel.hytale.server.core.cosmetics` package. A factory or a parent model object (e.g., a PlayerSkinPart) is responsible for its creation during the deserialization of a player's profile from the database.
- **Scope:** The lifetime of a PlayerSkinPartTexture instance is ephemeral. It exists only as part of a larger cosmetic data graph for a specific player. It is typically created when a player's data is loaded into memory and is discarded once that data is no longer needed (e.g., when the player logs out).
- **Destruction:** The object is managed by the Java Garbage Collector. There are no native resources or explicit cleanup methods. It is reclaimed once all references to it are dropped.

## Internal State & Concurrency
- **State:** The internal state, consisting of the texture path and an optional array of base colors, is set once during construction and is never modified thereafter. The object is effectively **immutable**.
- **Thread Safety:** This class is inherently **thread-safe**. Its immutable nature guarantees that multiple threads can safely read its state without locks or synchronization.

**WARNING:** While the object itself is immutable, the `getBaseColor` method returns a direct reference to the internal `String[]` array. Modifying the contents of this returned array is a severe anti-pattern that violates the object's immutability contract and can lead to unpredictable behavior in other threads.

## API Surface
The public API is minimal, providing read-only access to the deserialized data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getTexture() | String | O(1) | Returns the asset path for the texture. |
| getBaseColor() | String[] | O(1) | Returns the array of hex color codes, or null if not defined. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by feature developers. It is an internal component of the cosmetic deserialization pipeline. A parent object would create it while parsing a BSON document.

```java
// Hypothetical usage inside a parent data model
public class PlayerSkinPart {
    private PlayerSkinPartTexture mainTexture;

    public PlayerSkinPart(BsonDocument data) {
        if (data.containsKey("MainTexture")) {
            // The parent object is responsible for creating the child model
            BsonDocument textureDoc = data.getDocument("MainTexture");
            this.mainTexture = new PlayerSkinPartTexture(textureDoc);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The `protected` constructor prevents instantiation from outside the package. Attempting to bypass this (e.g., via reflection) would decouple the object from its BSON data source, creating an invalid state.
- **State Mutation:** Never modify the array returned by `getBaseColor`. This breaks the immutability guarantee and can cause rendering glitches or data corruption if the same object is referenced elsewhere.

```java
// DO NOT DO THIS
PlayerSkinPartTexture texture = cosmeticProfile.getHead().getTexture();
String[] colors = texture.getBaseColor();
if (colors != null) {
    colors[0] = "#FF0000"; // DANGEROUS: Mutates shared state
}
```

## Data Pipeline
PlayerSkinPartTexture acts as a deserialization endpoint in the player data loading process. It converts a specific sub-document from the database into a usable in-memory representation.

> Flow:
> MongoDB Document -> BSON Driver -> BsonDocument -> **new PlayerSkinPartTexture(doc)** -> Parent Cosmetic Model -> Game Logic (e.g., Player Appearance System)

