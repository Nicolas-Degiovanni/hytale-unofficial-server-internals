---
description: Architectural reference for PlayerSkinTintColor
---

# PlayerSkinTintColor

**Package:** com.hypixel.hytale.server.core.cosmetics
**Type:** Data Model / Transient

## Definition
```java
// Signature
public class PlayerSkinTintColor {
```

## Architecture & Concepts
The PlayerSkinTintColor class is a server-side, immutable data model that represents a single cosmetic color tint option for a player's skin. Its primary architectural role is to serve as a strongly-typed representation of configuration data deserialized from a BSON data source, most commonly a MongoDB collection.

This class is a leaf component within the server's cosmetic system. It does not orchestrate other components or contain complex logic. Instead, it encapsulates a piece of static data—an identifier and a set of color values—providing a safe and predictable data contract for higher-level services like a CosmeticManager or PlayerProfile service. By translating raw database documents into a dedicated Java object, it prevents primitive obsession and clarifies the data structure for the rest of the server application.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively within the `com.hypixel.hytale.server.core.cosmetics` package, as indicated by the `protected` constructor. A manager or factory class, such as a `CosmeticLoader`, is responsible for parsing a `BsonDocument` retrieved from the database and instantiating this object. Direct instantiation by application code is prohibited.
- **Scope:** The lifetime of a PlayerSkinTintColor instance is tied to its container. If it is part of a global cosmetic catalog loaded at server startup, it will persist for the entire server session. If it is loaded as part of a specific player's data, its scope is limited to that player's session.
- **Destruction:** As a simple Plain Old Java Object (POJO) with no external resource handles, it is managed entirely by the Java Garbage Collector. It is marked for collection once it is no longer referenced by any part of the application.

## Internal State & Concurrency
- **State:** The object's state is effectively immutable. The `id` and `baseColor` fields are populated once during construction and cannot be modified thereafter. This design guarantees that a PlayerSkinTintColor object is a stable data carrier throughout its lifecycle.

- **Thread Safety:** This class is inherently thread-safe due to its immutability. Multiple threads can safely access its data via the public getters without requiring any synchronization mechanisms like locks or volatile keywords.

    **Warning:** The `getBaseColor` method returns a direct reference to the internal `String[]` array. While the array reference itself cannot be changed, the *contents* of the array are mutable. Modifying this array is a critical anti-pattern that can lead to globally inconsistent state. Consumers must treat the returned array as read-only.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | String | O(1) | Returns the unique identifier for this tint color. |
| getBaseColor() | String[] | O(1) | Returns the array of color strings. The returned array should be treated as read-only. |

## Integration Patterns

### Standard Usage
This object is not intended to be created or managed directly. It is retrieved from a higher-level service that abstracts the data source.

```java
// Example: Retrieving a tint from a hypothetical cosmetic service
CosmeticService cosmeticService = server.getService(CosmeticService.class);
PlayerSkinTintColor fireTint = cosmeticService.getTintColor("fire_red");

if (fireTint != null) {
    String[] colors = fireTint.getBaseColor();
    // Use the color data for rendering or game logic
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use reflection or other means to bypass the `protected` constructor. The object's integrity depends on being created from a valid `BsonDocument` by its designated factory.
- **State Mutation:** Do not modify the array returned by `getBaseColor()`. This violates the immutability contract and can cause unpredictable behavior in other systems that share the same object instance.

```java
// ANTI-PATTERN: Modifying the internal state
PlayerSkinTintColor tint = cosmeticService.getTintColor("some_tint");
String[] colors = tint.getBaseColor();
colors[0] = "#FFFFFF"; // This corrupts the shared state for all users of this object
```

## Data Pipeline
PlayerSkinTintColor acts as a deserialization target in the data flow from the database to the game logic. It represents the point where unstructured BSON data is converted into a structured, type-safe Java object.

> Flow:
> MongoDB Collection -> BSON Driver -> `BsonDocument` -> **PlayerSkinTintColor (Constructor)** -> Cosmetic Catalog Cache -> Game Logic / Player Profile

