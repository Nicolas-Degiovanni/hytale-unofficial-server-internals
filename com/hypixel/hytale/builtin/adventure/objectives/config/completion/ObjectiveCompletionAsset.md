---
description: Architectural reference for ObjectiveCompletionAsset
---

# ObjectiveCompletionAsset

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.completion
**Type:** Configuration Asset

## Definition
```java
// Signature
public abstract class ObjectiveCompletionAsset {
```

## Architecture & Concepts

The ObjectiveCompletionAsset is an abstract base class that serves as a data-driven contract for defining the completion criteria of a quest objective in Hytale's adventure mode. It is a foundational component of the engine's polymorphic configuration system, allowing designers to specify diverse objective types in external data files (e.g., JSON) without requiring engine code changes.

This class itself contains no logic. Instead, it acts as a marker and a common ancestor for concrete implementations like *ObjectiveCompletionKillEntity* or *ObjectiveCompletionReachLocation*. The core architectural enabler is the static **CODEC** field, an instance of CodecMapCodec. This codec implements a type-dispatch pattern: it reads a "Type" field from an asset file and uses it as a key to look up the correct Java subclass to instantiate and deserialize the data into.

In essence, ObjectiveCompletionAsset is a schema for configuration, not a runtime component. It represents the "what" (e.g., kill 10 zombies) which is then interpreted by the "how" (the objective tracking systems during gameplay).

### Lifecycle & Ownership
-   **Creation:** Instances are never created directly using a constructor. They are instantiated exclusively by the Hytale serialization framework via the static **CODEC** field during the asset loading phase. This typically occurs when the game server or client loads a world's adventure mode configuration.
-   **Scope:** An instance of an ObjectiveCompletionAsset persists for the entire duration that its parent objective definition is loaded. As configuration data, it is effectively a static, in-memory representation of a file on disk and is considered immutable post-load.
-   **Destruction:** The object is garbage collected when the server shuts down or the game transitions to a state where the adventure mode assets are no longer required. Its lifecycle is tied to the Asset Manager, not to a specific gameplay session.

## Internal State & Concurrency
-   **State:** **Immutable**. Once an ObjectiveCompletionAsset is deserialized from a configuration file, its internal state is not expected to change. It is a read-only data container.
-   **Thread Safety:** **Thread-Safe**. Due to its immutable nature, a loaded asset can be safely read by multiple game systems on different threads without any need for locks or synchronization. For example, the quest logic thread and the UI thread can both reference the same asset instance concurrently without issue.

## API Surface

The primary public contract is not through instance methods but through the static codecs used for serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | CodecMapCodec | N/A | **Critical.** Static codec for polymorphic deserialization. Maps a "Type" identifier in a data file to a concrete Java subclass. |
| BASE_CODEC | BuilderCodec | N/A | Static base codec used by the framework to handle properties common to all completion assets. |

## Integration Patterns

### Standard Usage

A developer does not interact with this class directly at runtime. Instead, they define new objective completion types by extending ObjectiveCompletionAsset and registering the new subclass with the master codec. The engine handles the rest.

**1. Define a concrete subclass:**
```java
// In ObjectiveCompletionKill.java
public class ObjectiveCompletionKill extends ObjectiveCompletionAsset {
    public static final BuilderCodec<ObjectiveCompletionKill> CODEC = ...;
    private final String entityType;
    private final int count;
    // ... constructor, getters
}
```

**2. Register the new type with the master codec (typically during static initialization):**
```java
// In a central registry or the base class itself
ObjectiveCompletionAsset.CODEC.register("Kill", ObjectiveCompletionKill.CODEC);
```

**3. A game designer uses the new type in a JSON asset:**
```json
// In some_quest.json
{
  "objective": {
    "completionCondition": {
      "Type": "Kill",
      "entityType": "hytale:zombie",
      "count": 10
    }
  }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never attempt to create an instance via `new`. The entire system is designed to be driven by the codec framework, which guarantees correct initialization from data files.
-   **Runtime State Modification:** Do not attempt to modify the fields of a loaded asset. These objects represent static game data. Runtime progress should be stored in separate state-tracking objects, not in the configuration asset itself.
-   **Bypassing the CODEC:** Manually constructing asset objects in code circumvents the data-driven architecture. This creates brittle, hardcoded logic that is difficult for designers to modify.

## Data Pipeline

The ObjectiveCompletionAsset is the output of a deserialization pipeline that transforms on-disk configuration into in-memory Java objects.

> Flow:
> Adventure Quest JSON File -> Asset Loading System -> **ObjectiveCompletionAsset.CODEC** -> Instantiated Subclass (e.g., ObjectiveCompletionKill) -> Objective Definition Registry

