---
description: Architectural reference for TempAssetIdUtil
---

# TempAssetIdUtil

**Package:** com.hypixel.hytale.server.core.util
**Type:** Utility

## Definition
```java
// Signature
@Deprecated(forRemoval = true)
public class TempAssetIdUtil {
```

## Architecture & Concepts
TempAssetIdUtil is a deprecated static utility class that serves as a temporary, hardcoded repository for "magic string" asset identifiers. Its primary architectural role is to provide a centralized, albeit fragile, mapping between human-readable string constants (e.g., SOUND_EVENT_ITEM_REPAIR) and their corresponding integer indices within the runtime asset system.

This class represents a form of technical debt, created as a stop-gap solution before a fully dynamic and robust asset discovery and loading system was implemented. The presence of the `@Deprecated(forRemoval = true)` annotation is a critical warning; this component is scheduled for complete removal from the codebase. All new development must avoid dependencies on this class. Its existence is a transitional artifact, and its usage indicates a system that has not yet been migrated to the modern asset management pipeline.

## Lifecycle & Ownership
As a class containing only static members, TempAssetIdUtil does not follow a traditional object lifecycle.

- **Creation:** The class is loaded into the JVM by the ClassLoader upon its first reference. No instance is ever created.
- **Scope:** The class definition and its static members persist for the entire lifetime of the server's JVM.
- **Destruction:** It is unloaded when the JVM shuts down. There is no manual cleanup or destruction process.

## Internal State & Concurrency
- **State:** This class is entirely stateless. It consists of `public static final` constants, which are immutable, and a single pure static method that does not modify any state.
- **Thread Safety:** TempAssetIdUtil is inherently thread-safe. Its immutable constants and stateless method can be accessed from any thread without synchronization. The only side effect is logging, which is delegated to the thread-safe HytaleLogger.

## API Surface
The public contract consists of a collection of string constants and one utility method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSoundEventIndex(String) | int | O(1) | Translates a string sound event ID into its integer index. Returns 0 and logs a warning if the ID is not found. |

## Integration Patterns

### Standard Usage
The intended usage pattern is for legacy systems to resolve a hardcoded sound event string into a safe, usable integer index.

**WARNING:** This pattern is deprecated. New code must use the modern asset resolution system.

```java
// Legacy code retrieving a sound event index
int repairSoundIndex = TempAssetIdUtil.getSoundEventIndex(TempAssetIdUtil.SOUND_EVENT_ITEM_REPAIR);

// The sound system would then use this index
SoundSystem.playSound(repairSoundIndex);
```

### Anti-Patterns (Do NOT do this)
- **New Dependencies:** Do not introduce any new references to TempAssetIdUtil in the codebase. This directly increases technical debt and will break when the class is removed.
- **Relying on Fallback:** Do not write logic that depends on the fallback return value of 0 for non-existent sounds. This masks missing asset errors and can lead to silent failures or incorrect game behavior.
- **Extending the Class:** Do not add new constants to this class. All new assets must be registered and resolved through the proper AssetManager pipeline.

## Data Pipeline
The data flow for this component is a simple, one-way translation.

> Flow:
> Hardcoded String Constant -> **TempAssetIdUtil.getSoundEventIndex()** -> SoundEvent Asset Map Lookup -> Integer Index (or 0) -> Calling System

