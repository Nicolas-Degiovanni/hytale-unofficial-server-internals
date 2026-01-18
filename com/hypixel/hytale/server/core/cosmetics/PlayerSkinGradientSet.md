---
description: Architectural reference for PlayerSkinGradientSet
---

# PlayerSkinGradientSet

**Package:** com.hypixel.hytale.server.core.cosmetics
**Type:** Value Object / DTO

## Definition
```java
// Signature
public class PlayerSkinGradientSet {
```

## Architecture & Concepts
The PlayerSkinGradientSet class is an immutable data container that represents a single, named set of cosmetic gradients for a player skin. It acts as a Value Object within the server's cosmetic system.

Its primary role is to provide a strongly-typed, in-memory representation of cosmetic data that is deserialized from a BSON data source, such as a MongoDB database. This class encapsulates the raw data into a structured Java object, preventing direct manipulation of the underlying BSON documents throughout the rest of the application. It holds a collection of PlayerSkinPartTexture objects, mapping specific skin part names (e.g., *torso*, *left_arm*) to their corresponding gradient texture data.

This object is a leaf node in the larger player appearance data model. It does not contain any logic; its sole responsibility is to hold data in a predictable and safe format.

## Lifecycle & Ownership
- **Creation:** PlayerSkinGradientSet instances are created exclusively by a data loading or cosmetic management service within the same package. The `protected` constructor, which accepts a BsonDocument, is a clear indicator that instantiation is controlled and is part of a larger deserialization pipeline. A manager class is responsible for fetching the raw BSON and passing it to the constructor.
- **Scope:** The lifetime of a PlayerSkinGradientSet object is tied to the cosmetic data cache or the session of a specific player. Given its immutability, it is designed to be safely cached and shared, potentially for the entire lifetime of the server process if it represents a globally available cosmetic.
- **Destruction:** The object is managed by the Java garbage collector. It is eligible for destruction once all references from caches or player data objects are released.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields (`id`, `gradients`) are declared as `final` and are initialized only once within the constructor. The state of a PlayerSkinGradientSet cannot be changed after it has been created.
- **Thread Safety:** This class is **inherently thread-safe**. Due to its immutability, instances can be safely read by multiple threads concurrently without the need for external synchronization or locks. This makes it ideal for use in highly concurrent systems where cosmetic data must be accessed frequently from different parts of the game loop or network threads.

## API Surface
The public API is minimal and provides read-only access to the encapsulated data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | String | O(1) | Returns the unique identifier for this gradient set. |
| getGradients() | Map<String, PlayerSkinPartTexture> | O(1) | Returns the map of skin part names to their gradient textures. |

**WARNING:** While the returned Map interface defines mutation methods (e.g., put, remove), the underlying data structure is not intended to be modified. Altering the returned map is an unsupported operation and violates the immutable design of this class.

## Integration Patterns

### Standard Usage
A PlayerSkinGradientSet is not retrieved directly. It is typically accessed as part of a larger configuration object, such as a player's profile or a global cosmetic asset definition.

```java
// Example: Retrieving a gradient set from a player's cosmetic profile
PlayerCosmeticProfile profile = player.getCosmeticProfile();
PlayerSkin skin = profile.getActiveSkin();
PlayerSkinGradientSet gradients = skin.getGradientSet("fire_and_ice");

if (gradients != null) {
    String id = gradients.getId();
    // Use the gradient data for rendering or other logic
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create an instance using reflection or by modifying package access. These objects must only be created by the designated cosmetic data loader to ensure data integrity.
- **State Modification:** Do not attempt to modify the Map returned by getGradients. This class is designed to be a stable, immutable data source.

## Data Pipeline
This class exists at the end of a data deserialization pipeline. It transforms raw database records into a usable, type-safe Java object.

> Flow:
> MongoDB Document -> BSON Driver -> Cosmetic Data Service -> **PlayerSkinGradientSet(BsonDocument)** -> Player Profile / Cosmetic Cache -> Game Logic

