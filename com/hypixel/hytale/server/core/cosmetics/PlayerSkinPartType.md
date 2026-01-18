---
description: Architectural reference for PlayerSkinPartType
---

# PlayerSkinPartType

**Package:** com.hypixel.hytale.server.core.cosmetics
**Type:** Utility

## Definition
```java
// Signature
public enum PlayerSkinPartType {
```

## Architecture & Concepts
PlayerSkinPartType is a foundational enumeration that defines the canonical schema for a player avatar's customizable components. It serves as a static data contract, ensuring that different engine subsystems—such as the cosmetics manager, rendering pipeline, and network serialization—all refer to avatar parts using a consistent, type-safe vocabulary.

This enum is not a service or an active component; rather, it is a core data model that underpins the entire player customization system. By centralizing the definition of skin parts, it prevents data corruption and logic errors that could arise from using string-based or integer-based identifiers. It is the "source of truth" for what constitutes a modifiable part of a player model.

## Lifecycle & Ownership
- **Creation:** Instances of this enum are created and managed exclusively by the Java Virtual Machine (JVM). They are instantiated automatically when the PlayerSkinPartType class is loaded by the class loader, typically upon its first reference in the code.
- **Scope:** All enum constants (Eyes, Haircut, etc.) are singletons that persist for the entire lifetime of the server application. They are globally accessible and immutable.
- **Destruction:** The enum and its constants are unloaded only when the JVM shuts down. There is no manual memory management or cleanup required.

## Internal State & Concurrency
- **State:** PlayerSkinPartType is a stateless and immutable data type. Each enum constant is a fixed value with no mutable fields.
- **Thread Safety:** This enum is inherently thread-safe. As immutable singletons managed by the JVM, its constants can be safely accessed and passed between any number of concurrent threads without synchronization or locking mechanisms.

## API Surface
The primary API consists of the predefined enum constants. Standard Java enum methods like `values()` and `valueOf(String)` are also available.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Eyes | PlayerSkinPartType | O(1) | Represents the eye component of the avatar. |
| Ears | PlayerSkinPartType | O(1) | Represents the ear component of the avatar. |
| Mouth | PlayerSkinPartType | O(1) | Represents the mouth component of the avatar. |
| ... | ... | ... | ... |
| Gloves | PlayerSkinPartType | O(1) | Represents the glove or hand accessory component. |

## Integration Patterns

### Standard Usage
This enum is typically used as a key in data structures or as a parameter to identify which part of a player's skin is being modified or requested.

```java
// Example: Storing a player's cosmetic choices
Map<PlayerSkinPartType, AssetReference> playerCosmetics = new EnumMap<>(PlayerSkinPartType.class);
playerCosmetics.put(PlayerSkinPartType.Haircut, new AssetReference("hytale:long_hair"));

// Example: Using in a switch statement for logic
public void applyCosmetic(PlayerSkinPartType part, AssetReference asset) {
    switch (part) {
        case Haircut:
            player.model.setHair(asset);
            break;
        case Shoes:
            player.model.setFootwear(asset);
            break;
        // ... other cases
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Serialization by Ordinal:** Do not use the `ordinal()` method to serialize or store this enum value in a database or send it over the network. The integer value is fragile and will change if the enum declaration order is modified, leading to catastrophic data corruption. Always use the `name()` method for a stable, string-based representation.
- **Extensibility Assumptions:** Do not write code that assumes the set of skin parts is final. While this enum defines the core set, future updates or mods may add to it. Avoid hardcoded array sizes based on `values().length`.

## Data Pipeline
PlayerSkinPartType acts as a data identifier that flows through various systems. It does not process data itself but gives context to the data it accompanies.

> Flow:
> Player Customization UI -> **PlayerSkinPartType** + Asset ID -> Network Packet -> Server Logic -> Player Data Storage -> Other Game Clients -> Renderer

