---
description: Architectural reference for PlayerSkinGradient
---

# PlayerSkinGradient

**Package:** com.hypixel.hytale.server.core.cosmetics
**Type:** Data Model / DTO

## Definition
```java
// Signature
public class PlayerSkinGradient extends PlayerSkinTintColor {
```

## Architecture & Concepts
The PlayerSkinGradient class is a specialized, immutable data model representing a cosmetic option that applies a textured gradient to a player's skin. It is a direct extension of the PlayerSkinTintColor, inheriting base color properties while adding a specific texture component.

This class serves as a concrete, in-memory representation of a record from the cosmetics database. Its existence and structure are dictated by the BSON data format, indicating it is part of the data persistence and deserialization layer. Architecturally, it acts as a read-only data carrier, transporting cosmetic configuration from the server's database to the game logic that needs to apply it. It is not a service and contains no business logic; its sole responsibility is to hold data.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by an internal data access layer or factory during the deserialization of a BsonDocument, which is typically retrieved from a MongoDB database. The constructor is marked as protected to prevent direct, uncontrolled instantiation.
- **Scope:** The lifetime of a PlayerSkinGradient object is transient and scoped to the lifetime of the parent object that holds it, such as a player's loaded profile or cosmetic inventory.
- **Destruction:** The object is managed by the Java Garbage Collector and is destroyed when no longer referenced. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The internal state of PlayerSkinGradient is **immutable**. All fields, including the inherited ones, are set once during construction from the BsonDocument and cannot be modified thereafter. This makes the object a reliable, point-in-time snapshot of the database record.
- **Thread Safety:** Due to its immutable nature, PlayerSkinGradient is inherently **thread-safe**. It can be safely shared and read by multiple threads without locks or other synchronization primitives.

## API Surface
The public contract is minimal, exposing only data accessors.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getTexture() | String | O(1) | Returns the identifier for the gradient texture. Returns null if not present in the source data. |

## Integration Patterns

### Standard Usage
This object should never be created directly. It is intended to be retrieved from a higher-level manager or data object and consumed as a read-only data source.

```java
// Correctly retrieve the cosmetic data from a player's profile
PlayerCosmetics cosmetics = player.getActiveCosmetics();
PlayerSkinGradient gradient = cosmetics.getSkinGradient();

// Use the data for rendering or logic
if (gradient != null) {
    String textureId = gradient.getTexture();
    // ... apply texture to the player model
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The protected constructor explicitly forbids creation via `new PlayerSkinGradient()`. Attempting to bypass this via reflection will lead to an unsupported and unstable state. The object must be sourced from the data layer.
- **State Assumption:** Do not assume the `texture` field is always non-null. The source BsonDocument may not contain the "Texture" key, in which case `getTexture()` will return null. Always perform a null check.

## Data Pipeline
PlayerSkinGradient sits at the end of the data loading pipeline, acting as the bridge between raw database records and the server's runtime game logic.

> Flow:
> MongoDB Record (BSON) -> BSON Deserializer -> **PlayerSkinGradient** (In-Memory DTO) -> Player Profile Service -> Game Logic / Renderer

