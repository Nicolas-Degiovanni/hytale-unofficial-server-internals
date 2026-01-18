---
description: Architectural reference for CosmeticType
---

# CosmeticType

**Package:** com.hypixel.hytale.server.core.cosmetics
**Type:** Type-Safe Enumeration

## Definition
```java
// Signature
public enum CosmeticType {
```

## Architecture & Concepts
The CosmeticType enum is a foundational data structure that establishes a fixed, compile-time vocabulary for all character customization categories within the Hytale engine. It serves as a canonical and type-safe discriminator, eliminating the ambiguity and risks associated with string-based identifiers (e.g., "HAIRCUTS" vs "haircuts").

This enumeration is a core component of the player identity and appearance systems. It is used extensively to:
- Categorize cosmetic items in game assets and databases.
- Key data in player profile and inventory maps.
- Drive logic in the character creation UI.
- Inform the rendering pipeline which character model parts to attach or modify.

By defining a closed set of possible cosmetic slots, CosmeticType enforces data integrity and simplifies logic throughout the codebase. Any system that deals with player appearance will directly or indirectly reference these constants.

## Lifecycle & Ownership
- **Creation:** Instances of this enum are created automatically by the Java Virtual Machine (JVM) during class loading. This typically occurs the first time the CosmeticType class is referenced by any part of the server code. There is no manual instantiation.
- **Scope:** All enum constants (e.g., EMOTES, SKIN_TONES) are static, final instances that persist for the entire lifetime of the server application. They are effectively global singletons.
- **Destruction:** The enum and its constants are unloaded from memory only when the JVM shuts down. There is no manual cleanup or garbage collection of the enum instances themselves.

## Internal State & Concurrency
- **State:** CosmeticType is **immutable**. Its state is the predefined set of constants, which cannot be altered at runtime. Each constant holds its name and its ordinal value, managed internally by the JVM.
- **Thread Safety:** This enumeration is inherently **thread-safe**. As immutable singletons, its constants can be safely accessed and passed between any number of concurrent threads without requiring locks or synchronization. This is a critical property for a high-performance, multi-threaded server environment.

## API Surface
The primary API consists of the predefined constants themselves. Standard methods from the java.lang.Enum base class, such as *values()* and *valueOf(String)*, are also available but should be used with caution due to performance implications (e.g., *values()* creates a new array on each call).

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| EMOTES | CosmeticType | O(1) | Represents the category for player emotes. |
| SKIN_TONES | CosmeticType | O(1) | Represents the category for player skin colors. |
| HAIRCUTS | CosmeticType | O(1) | Represents the category for player hairstyles. |
| ... | ... | ... | ...and so on for all 24 defined types. |

## Integration Patterns

### Standard Usage
CosmeticType is most often used as a discriminator in control flow statements (switch) or as a key in data structures like an EnumMap for efficient, type-safe lookups.

```java
// How a developer should normally use this
import java.util.Map;
import java.util.EnumMap;

// Using the enum as a key for a player's equipped items
Map<CosmeticType, AssetKey> equippedCosmetics = new EnumMap<>(CosmeticType.class);
equippedCosmetics.put(CosmeticType.HAIRCUTS, new AssetKey("hytale:elf_ponytail"));
equippedCosmetics.put(CosmeticType.CAPES, new AssetKey("hytale:adventurer_cape"));

// Using the enum to drive logic
public void applyCosmetic(Player player, CosmeticType type, AssetKey asset) {
    switch (type) {
        case HAIRCUTS:
            player.getAppearance().setHair(asset);
            break;
        case CAPES:
            player.getAppearance().setCape(asset);
            break;
        // ... other cases
        default:
            log.warn("Unhandled cosmetic type: " + type);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Ordinal Reliance:** Do not persist or serialize the ordinal value of an enum constant. The integer value is fragile and will change if the declaration order in the source file is modified, leading to catastrophic data corruption. Always use the *name()* for serialization.
- **String Comparison:** Do not compare enum constants by converting them to strings. Use direct object comparison with ==, which is both safe and highly performant for enums.
- **Extensibility:** Do not attempt to extend this enum. Its nature as a closed, fixed set is its primary architectural strength. If new types are needed, they must be added directly to the source file.

## Data Pipeline
CosmeticType acts as a routing key or label for data as it moves through various systems. It provides context to otherwise generic asset or item identifiers.

> Flow:
> Game Asset Manifest -> Asset Loader -> **CosmeticType** (as metadata) -> Player Inventory System -> Character Renderer


