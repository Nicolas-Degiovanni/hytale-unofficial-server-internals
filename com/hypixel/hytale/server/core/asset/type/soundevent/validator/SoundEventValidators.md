---
description: Architectural reference for SoundEventValidators
---

# SoundEventValidators

**Package:** com.hypixel.hytale.server.core.asset.type.soundevent.validator
**Type:** Utility

## Definition
```java
// Signature
public class SoundEventValidators {
    // Static fields and nested classes
    
    public static class ChannelValidator implements Validator<String> {
        // ...
    }

    public static class LoopValidator implements Validator<String> {
        // ...
    }
}
```

## Architecture & Concepts

The **SoundEventValidators** class is a static utility that provides a collection of pre-configured, reusable **Validator** instances for the Hytale asset system. It serves as a central registry for validation logic specific to **SoundEvent** assets, ensuring that sound events referenced in other configuration files adhere to specific technical constraints, such as being mono, stereo, looping, or one-shot.

This class fits into the broader Hytale *Codec and Schema* framework. During asset loading and parsing, the codec system can be configured to apply validators to specific fields. When a field containing a sound event name (a String) is processed, the codec invokes the configured validator from this class.

The core validation mechanism involves two steps:
1.  Resolving the sound event name (a String) into a full **SoundEvent** object by querying the global **SoundEvent.getAssetMap()**.
2.  Inspecting the properties of the resolved **SoundEvent** object to check if it meets the required criteria (e.g., channel count, presence of a looping layer).

The class also exposes several **ValidatorCache** instances. These wrappers provide a memoization layer over the core validators, significantly improving performance in scenarios where the same sound event is validated repeatedly across many different assets.

## Lifecycle & Ownership

-   **Creation:** This class is a static utility and is never instantiated. Its public static fields (**LOOPING**, **ONESHOT**, **MONO**, **STEREO**, and associated caches) are initialized by the JVM during class loading. This typically occurs once at server startup.
-   **Scope:** The validator instances are static singletons that persist for the entire lifetime of the server application.
-   **Destruction:** The objects are eligible for garbage collection only when the application's class loader is unloaded, which effectively means at server shutdown.

## Internal State & Concurrency

-   **State:** The individual **Validator** instances (**LoopValidator**, **ChannelValidator**) are effectively immutable. Their validation criteria (e.g., `channelCount`, `looping`) are final fields set during static initialization. However, the validation process itself is state-dependent, as it relies on the global, mutable state of the **SoundEvent.getAssetMap()**. The exposed **ValidatorCache** fields are stateful by design, as they store the results of previous validation operations.

-   **Thread Safety:** The validators are conditionally thread-safe. The `accept` methods do not modify internal state and can be called concurrently. However, safety is critically dependent on the thread safety of the underlying **SoundEvent.getAssetMap()**.

    **WARNING:** The validation process assumes that the **SoundEvent.getAssetMap()** is fully populated and in a read-only state when validation occurs. If another thread modifies the asset map while a validator is executing, behavior is undefined and may result in a **ConcurrentModificationException** or incorrect validation results. Asset loading and validation should be strictly ordered during the server bootstrap sequence. The **ValidatorCache** instances are assumed to be internally synchronized and safe for concurrent use.

## API Surface

The primary API consists of the public static fields which provide access to pre-configured validator instances. Direct instantiation of the nested classes is discouraged.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| LOOPING | LoopValidator | O(N) | A validator that fails if the referenced **SoundEvent** does not contain at least one looping layer. N is the number of layers. |
| ONESHOT | LoopValidator | O(N) | A validator that fails if the referenced **SoundEvent** contains any looping layers. N is the number of layers. |
| MONO | ChannelValidator | O(1) | A validator that fails if the referenced **SoundEvent** does not have exactly 1 audio channel. |
| STEREO | ChannelValidator | O(1) | A validator that fails if the referenced **SoundEvent** does not have exactly 2 audio channels. |
| MONO_VALIDATOR_CACHE | ValidatorCache | O(1) | A cached version of the **MONO** validator for improved performance. |
| STEREO_VALIDATOR_CACHE | ValidatorCache | O(1) | A cached version of the **STEREO** validator for improved performance. |
| ONESHOT_VALIDATOR_CACHE | ValidatorCache | O(1) | A cached version of the **ONESHOT** validator for improved performance. |

## Integration Patterns

### Standard Usage

These validators are not intended to be invoked directly in game logic. Instead, they are referenced declaratively within the Hytale schema configuration. The codec system uses these references to enforce rules during asset deserialization.

```java
// Hypothetical example of configuring a schema for a game object
// that requires a one-shot footstep sound.

SchemaDefinition definition = new SchemaDefinition();

// The "footstepSound" field must reference a valid SoundEvent that is
// non-looping (a one-shot).
definition.addField("footstepSound", String.class)
    .withValidator(SoundEventValidators.ONESHOT);

// The codec system will now automatically apply this validation
// when loading any asset that uses this schema.
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not create new instances of **ChannelValidator** or **LoopValidator**. The provided static singletons are sufficient and enable performance benefits from caching.
    ```java
    // ANTI-PATTERN
    Validator<String> myValidator = new SoundEventValidators.ChannelValidator(1); 
    
    // CORRECT
    Validator<String> correctValidator = SoundEventValidators.MONO;
    ```

-   **Premature Validation:** Do not attempt to use these validators before the server's asset loading phase is complete. The validators rely on a fully populated **SoundEvent.getAssetMap()**. Using them prematurely will cause legitimate sound events to fail validation because they have not been loaded into the map yet.

## Data Pipeline

The flow of data through a validator is driven by the parent codec system during asset parsing.

> Flow:
> Asset File (e.g., JSON) -> Codec Deserializer -> **Validator.accept(soundEventName)** -> Global AssetMap Lookup -> **SoundEvent Property Check** -> Mutate ValidationResults Object

