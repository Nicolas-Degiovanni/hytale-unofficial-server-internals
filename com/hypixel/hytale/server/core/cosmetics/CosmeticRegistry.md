---
description: Architectural reference for CosmeticRegistry
---

# CosmeticRegistry

**Package:** com.hypixel.hytale.server.core.cosmetics
**Type:** Transient

## Definition
```java
// Signature
public class CosmeticRegistry {
```

## Architecture & Concepts
The CosmeticRegistry is a foundational, read-only data repository responsible for loading and indexing all player cosmetic definitions from the game's core asset files. It acts as an in-memory database for cosmetic items, deserializing BSON-formatted JSON files into strongly-typed Java objects at application startup.

Its primary architectural purpose is to decouple game systems from the underlying asset file system. By loading all cosmetic data once into a structured, immutable cache, it provides high-performance, key-based access to systems like the Character Creator, Player Renderer, and monetization services. This eliminates the need for repeated, expensive file I/O and parsing operations during active gameplay.

The registry is organized by cosmetic category, with each category represented by a dedicated, unmodifiable Map. This design ensures predictable and safe data access across the application.

## Lifecycle & Ownership
- **Creation:** An instance is created once during the server or client bootstrap sequence. It requires a valid AssetPack as a constructor dependency, from which it reads all cosmetic definition files. It is a heavyweight object and its instantiation is a significant part of the application's initial loading phase.
- **Scope:** The object's lifetime is tied directly to the application session. It is designed to be a long-lived service that persists until the server or client is shut down.
- **Destruction:** The CosmeticRegistry holds no unmanaged resources and has no explicit destruction or close method. It is reclaimed by the Java garbage collector when the application context that created it is terminated.

## Internal State & Concurrency
- **State:** The internal state is **effectively immutable**. Upon construction, the registry populates over twenty internal Map fields from disk. The private load method immediately wraps each map in `Collections.unmodifiableMap`, programmatically preventing any subsequent modification. This guarantees that the registry's data remains consistent throughout its lifecycle.
- **Thread Safety:** This class is **inherently thread-safe**. Its immutable design ensures that multiple threads can read cosmetic data concurrently without any risk of race conditions or data corruption. No external synchronization or locking is required when accessing its methods.

## API Surface
The public API consists primarily of direct accessors for each cosmetic category. The generic getByType method provides a more flexible, type-driven access pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getByType(type) | Map<String, ?> | O(1) | Returns the unmodifiable map for the specified CosmeticType. Central dispatcher for cosmetic data. |
| getEmotes() | Map<String, Emote> | O(1) | Returns the read-only map of all registered emotes, keyed by their string ID. |
| getHaircuts() | Map<String, PlayerSkinPart> | O(1) | Returns the read-only map of all registered haircuts, keyed by their string ID. |

## Integration Patterns

### Standard Usage
The CosmeticRegistry should be treated as a shared service, retrieved from a dependency injection container or service locator. It should never be instantiated directly within game logic.

```java
// Correct: Retrieve the singleton-like instance from a central context
CosmeticRegistry registry = serverContext.getService(CosmeticRegistry.class);

// Access a specific category of cosmetics
Map<String, PlayerSkinPart> availableHaircuts = registry.getHaircuts();
PlayerSkinPart buzzcut = availableHaircuts.get("buzzcut_male_01");

// Or, use the generic dispatcher
Map<String, ?> capesMap = registry.getByType(CosmeticType.CAPES);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new CosmeticRegistry(assetPack)` in performance-critical code or general game logic. Instantiation involves significant disk I/O and parsing, and is only intended to be performed once at startup.
- **Attempted Modification:** The maps returned by this registry are unmodifiable. Attempting to add, remove, or clear elements will result in an `UnsupportedOperationException`. The registry is a read-only source of truth.
- **Error Prone Casting:** When using `getByType`, be certain of the expected value type to avoid a `ClassCastException` at runtime.

## Data Pipeline
The CosmeticRegistry is the endpoint of a critical data loading pipeline that transforms raw asset files into a queryable, in-memory data structure.

> Flow:
> AssetPack Files (`Cosmetics/CharacterCreator/*.json`) -> Filesystem Read -> BSON Parser -> **CosmeticRegistry** -> Unmodifiable In-Memory Maps -> Game Systems (Character Creator, Player Renderer)

