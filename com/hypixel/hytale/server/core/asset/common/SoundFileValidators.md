---
description: Architectural reference for SoundFileValidators
---

# SoundFileValidators

**Package:** com.hypixel.hytale.server.core.asset.common
**Type:** Utility

## Definition
```java
// Signature
public class SoundFileValidators {
    // ... static fields and methods
    
    public static class ChannelValidator implements Validator<String> {
        // ...
    }
}
```

## Architecture & Concepts
The SoundFileValidators class is a specialized component within the Hytale asset processing and validation pipeline. It does not function as a standalone service but rather as a collection of validation rules used by the broader schema and codec system. Its primary responsibility is to enforce audio format constraints, specifically the channel count (mono or stereo), for Ogg Vorbis sound files referenced in asset definitions.

Architecturally, this class acts as a bridge between declarative asset schemas and the low-level state of audio files. It decouples the asset definition from the direct inspection of file metadata by leveraging an intermediate caching layer, OggVorbisInfoCache. This design ensures that validation is both fast and consistent, as it operates on pre-processed, cached metadata rather than performing expensive file I/O during the validation step.

The core logic resides within the nested ChannelValidator class, which implements the engine's standard Validator interface. This allows it to be seamlessly integrated into any schema that defines a string field representing a path to a sound asset.

## Lifecycle & Ownership
- **Creation:** The SoundFileValidators class itself is never instantiated. Its static validator instances, MONO and STEREO, are initialized once upon class loading by the JVM. Custom ChannelValidator instances can be created, but this is an atypical use case.
- **Scope:** The static MONO and STEREO validators are application-scoped and persist for the entire lifetime of the server or client process. They are effectively global singletons.
- **Destruction:** The static instances are eligible for garbage collection only when the SoundFileValidators class is unloaded by the JVM, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** The SoundFileValidators utility class is stateless. The nested ChannelValidator holds a single, immutable `channelCount` field, making its instances effectively immutable after construction. This class does not maintain any internal caches or mutable state; it is a pure-logic component that relies on external systems for stateful information (OggVorbisInfoCache).

- **Thread Safety:** This class is fully thread-safe. The validator instances are immutable, and the core `accept` method is re-entrant. It can be safely used by multiple threads simultaneously during a parallel asset validation process without requiring any external locking. Its dependency, OggVorbisInfoCache, is expected to be a thread-safe cache implementation.

## API Surface
The public contract is minimal, exposing pre-configured validators and a helper method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| MONO | ChannelValidator | O(1) | A static, shared validator instance that enforces a single audio channel. |
| STEREO | ChannelValidator | O(1) | A static, shared validator instance that enforces two audio channels. |
| getEncoding(int) | String | O(1) | A static helper method to convert a channel count into a human-readable string. |
| ChannelValidator.accept(String, ValidationResults) | void | O(1) | Executes the validation logic. Complexity assumes a cache hit in OggVorbisInfoCache. |

## Integration Patterns

### Standard Usage
This validator is not intended to be invoked directly in game logic. It is designed to be attached to a field within an asset schema definition. The schema processing system then invokes it automatically.

The primary integration is declarative, likely within a JSON or HOCON asset file that references one of the static instances. A programmatic example of a custom validation pass would look like this:

```java
// Assume 'results' is a valid ValidationResults object
// and 'assetPath' is the string from the asset file.
String assetPath = "sounds/player/footstep_grass1.ogg";
ValidationResults results = new ValidationResults();

// Using the pre-configured static validator
SoundFileValidators.MONO.accept(assetPath, results);

if (results.hasFailed()) {
    // Handle validation failure
    System.out.println("Validation failed: " + results.getMessages());
}
```

### Anti-Patterns (Do NOT do this)
- **Redundant Instantiation:** Do not create new instances for standard channel counts. This is wasteful and bypasses the use of the shared, static singletons.
  ```java
  // AVOID THIS
  Validator<String> badValidator = new SoundFileValidators.ChannelValidator(1);
  
  // PREFER THIS
  Validator<String> goodValidator = SoundFileValidators.MONO;
  ```
- **Pre-emptive Validation:** Do not attempt to use this validator on a file path before the asset has been processed and its metadata has been populated into the OggVorbisInfoCache. Doing so will always result in a "No such ogg file" failure, even if the file exists on disk, because the validator *only* checks the cache.

## Data Pipeline
The validator operates as a single step within a larger asset ingestion and validation pipeline. It does not transform data but rather asserts conditions on it.

> Flow:
> Asset Definition (e.g., JSON) -> Schema Deserializer -> **SoundFileValidators.ChannelValidator** -> ValidationResults -> Asset Loading Decision

