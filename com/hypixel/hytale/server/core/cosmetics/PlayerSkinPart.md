---
description: Architectural reference for PlayerSkinPart
---

# PlayerSkinPart

**Package:** com.hypixel.hytale.server.core.cosmetics
**Type:** Data Model / DTO

## Definition
```java
// Signature
public class PlayerSkinPart {
```

## Architecture & Concepts

The PlayerSkinPart class is a data model that represents a single, configurable component of a player's cosmetic appearance. It serves as the in-memory representation of a cosmetic item defined in an external data source, such as a database or a game asset file. Examples of a PlayerSkinPart include a specific hairstyle, a shirt, a hat, or a pair of shoes.

This class is fundamentally a data-driven container, designed to be deserialized from a BSON document. Its primary role is to hold the structured properties of a cosmetic item, such as its unique identifier, model path, texture information, and behavioral tags. It is a core component of the server's character customization system.

The architecture supports complex cosmetic definitions through two key mechanisms:
1.  **Texture Maps:** A single part can define multiple named textures, allowing for different color schemes or patterns on the same 3D model.
2.  **Variants:** A part can contain a map of Variant objects, where each variant can specify an entirely different model and its own set of textures. This allows a single logical item, like "Knight Helmet", to have multiple distinct visual appearances, such as "Open Visor" and "Closed Visor", without requiring separate top-level asset definitions.

The server's cosmetic and player appearance services consume these objects to assemble a complete description of a player's character, which is then used for server-side logic and ultimately serialized to clients for rendering.

### Lifecycle & Ownership
-   **Creation:** A PlayerSkinPart is instantiated exclusively through its protected constructor, which accepts a BsonDocument. This process is managed by a higher-level system, such as a CosmeticManager or an asset loader, during the deserialization of game data. It is never created directly in game logic.
-   **Scope:** These objects represent canonical game assets. Once loaded from the data source, they are typically held in a server-wide cache for the entire duration of the server's runtime. This avoids the high cost of repeated BSON parsing and object allocation.
-   **Destruction:** The object is eligible for garbage collection when the server shuts down or if the central cosmetic asset cache is explicitly cleared. Its lifecycle is tied to the asset management system, not to any individual player session.

## Internal State & Concurrency
-   **State:** The state of a PlayerSkinPart is **effectively immutable**. All internal fields are private and are populated only once within the constructor. There are no public setter methods, making the object a read-only data container after its creation.
-   **Thread Safety:** The class is **conditionally thread-safe**. Because its internal state does not change after construction, it can be safely read by multiple threads simultaneously, which is essential for a shared server-wide cache.

    **WARNING:** The methods getTextures and getVariants return direct references to the internal Map objects. While the class itself does not modify these maps, a consumer could theoretically mutate the returned collections. Doing so would violate the immutability contract and corrupt the cached asset state for all threads on the server. The returned collections must be treated as read-only.

## API Surface

The public API consists entirely of accessor methods to retrieve the deserialized cosmetic data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | String | O(1) | Returns the unique, non-human-readable identifier for the part. |
| getName() | String | O(1) | Returns the human-readable display name of the part. |
| getModel() | String | O(1) | Returns the asset path for the default 3D model. |
| getTextures() | Map | O(1) | Returns the map of texture definitions for the default model. |
| getVariants() | Map | O(1) | Returns the map of alternative variants for this part. |
| getHairType() | HaircutType | O(1) | Returns an enum classifying the part's interaction with hair rendering. |
| getHeadAccessoryType() | HeadAccessoryType | O(1) | Returns an enum classifying how the part covers the head. |
| isDefaultAsset() | boolean | O(1) | Indicates if this is a default part available to all players. |

## Integration Patterns

### Standard Usage

This class is not intended to be instantiated or managed directly. It is retrieved from a central service that holds the cache of all loaded cosmetic assets. Game logic queries this service to get the data needed to describe a player's appearance.

```java
// Example: Retrieving a skin part from a hypothetical CosmeticService
CosmeticService cosmeticService = serverContext.getService(CosmeticService.class);
PlayerSkinPart hairstyle = cosmeticService.getSkinPartById("hair_long_01");

if (hairstyle != null) {
    String modelPath = hairstyle.getModel();
    // ... use the model path for player assembly
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never attempt to create a PlayerSkinPart using reflection or by manually constructing a BsonDocument. These objects must only be created by the server's official asset loading pipeline to ensure data integrity.
-   **State Mutation:** Do not modify the Map instances returned by getVariants() or getTextures(). This will lead to unpredictable behavior and server-wide state corruption, as the object is shared across all threads.
-   **Reliance on Name:** Do not use getName() for programmatic lookups. The name is for display purposes only and may change. Always use getId() for reliable identification.

## Data Pipeline

The PlayerSkinPart object is a key step in the pipeline that transforms raw cosmetic data from storage into a usable in-memory representation for the game server.

> Flow:
> BSON Data Source (e.g., MongoDB) -> BSON Deserializer -> **new PlayerSkinPart(doc)** -> CosmeticManager Cache -> Player Appearance Service -> Network Serialization -> Game Client

