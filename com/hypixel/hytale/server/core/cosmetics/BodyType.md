---
description: Architectural reference for BodyType
---

# BodyType

**Package:** com.hypixel.hytale.server.core.cosmetics
**Type:** Enum / Singleton

## Definition
```java
// Signature
public enum BodyType {
```

## Architecture & Concepts
The BodyType enum serves as a fundamental, type-safe discriminator for character models and associated cosmetic assets within the game engine. It provides a constrained set of identifiers for the primary skeletal and mesh archetypes available for player avatars.

This enumeration is a critical component of the data model for player characters, cosmetics, and armor. By using an enum instead of primitive types like integers or strings, the system guarantees at compile time that only valid body types can be referenced, preventing a wide class of data corruption and runtime errors. It acts as a primary key for lookups in various asset management and rendering systems to select the appropriate 3D model, texture set, or animation graph.

## Lifecycle & Ownership
- **Creation:** Instances of BodyType are constructed automatically by the Java Virtual Machine (JVM) during class loading. This process is guaranteed to happen only once.
- **Scope:** As a static component of the class, BodyType instances exist for the entire lifetime of the application's ClassLoader. They are effectively permanent, global singletons.
- **Destruction:** Instances are garbage collected only when the defining ClassLoader is unloaded, an event that typically coincides with application shutdown.

## Internal State & Concurrency
- **State:** BodyType instances are deeply immutable. Their state consists solely of their declared name (e.g., Masculine) and ordinal, which are fixed at compile time.
- **Thread Safety:** This enum is unconditionally thread-safe. Its instances are constants and can be safely accessed and passed between any number of threads without synchronization. This is a core guarantee of the Java language specification for enums.

## API Surface
The primary API consists of the constants themselves. Standard enum methods like `values()` and `valueOf(String)` are also available but are not listed here.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Masculine | BodyType | O(1) | Represents the masculine character model archetype. |
| Feminine | BodyType | O(1) | Represents the feminine character model archetype. |

## Integration Patterns

### Standard Usage
BodyType should be used as a field within data transfer objects (DTOs) or entity components to specify the model variation. Logic should use direct equality checks or switch statements for branching.

```java
// Example of using BodyType in a player data component
public void applyCosmetic(Player player, Cosmetic cosmetic) {
    BodyType playerBody = player.getProfile().getBodyType();

    // Select the correct asset based on the player's body type
    Asset cosmeticAsset = cosmetic.getAssetFor(playerBody);

    if (cosmeticAsset != null) {
        player.getRenderer().apply(cosmeticAsset);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Serialization by Ordinal:** Do not persist the enum's ordinal value (`BodyType.Masculine.ordinal()`). If the enum declaration order changes in the future, all persisted data will become corrupt. Persist the name (`BodyType.Masculine.name()`) instead.
- **String Comparison:** Avoid comparing the enum's string representation. Use direct object equality, which is both safer and significantly more performant.
    - **BAD:** `if (bodyType.toString().equals("Masculine"))`
    - **GOOD:** `if (bodyType == BodyType.Masculine)`

## Data Pipeline
As a data model primitive, BodyType does not process data itself but is a critical piece of data that flows through other systems.

> Flow:
> Player Database -> Profile Deserializer -> Player Entity Component -> **BodyType** field -> Rendering System -> Asset Lookup

