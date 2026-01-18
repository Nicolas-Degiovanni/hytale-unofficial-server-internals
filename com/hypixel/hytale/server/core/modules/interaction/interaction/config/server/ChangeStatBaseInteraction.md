---
description: Architectural reference for ChangeStatBaseInteraction
---

# ChangeStatBaseInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server
**Type:** Abstract Configuration Model

## Definition
```java
// Signature
public abstract class ChangeStatBaseInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The ChangeStatBaseInteraction class is an abstract base for data-driven interaction configurations. It is not an active service or manager; rather, it serves as a static data model representing a single, configurable action that modifies an entity's statistics. These configurations are deserialized from server asset files (e.g., JSON) at startup.

Its primary architectural role is to bridge the gap between human-readable configuration and the server's high-performance EntityStatsModule. This is achieved through its static **CODEC** field, which defines the schema, validation rules, and post-processing logic for this interaction type.

A key feature is the two-stage representation of stat modifiers:
1.  **entityStatAssets:** A map of string-based stat names to float values (e.g., "hytale:health" -> 20.0f). This is the direct representation from the configuration file.
2.  **entityStats:** A map of integer-based stat IDs to float values. This is a resolved, runtime-optimized representation created during the `afterDecode` lifecycle hook.

This resolution from string names to integer IDs is a critical performance optimization, allowing the game server to avoid expensive string comparisons and lookups during gameplay when the interaction is executed.

## Lifecycle & Ownership
-   **Creation:** Instances are not created directly using a constructor. They are exclusively instantiated by the Hytale **Codec** system when the server loads and parses configuration assets. The static `CODEC` field is the factory and deserializer for all subclasses.
-   **Scope:** An instance of ChangeStatBaseInteraction persists for the entire server session. Its lifetime is tied to the asset that defines it (e.g., an item definition, a world trigger). It is loaded once and held in memory.
-   **Destruction:** The object is eligible for garbage collection when the server shuts down or performs a full asset reload that discards the old configuration. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** This object is stateful, containing the parameters of the stat change. However, it is designed to be **effectively immutable** after its creation by the codec system. All its fields are populated during deserialization and are not expected to change during runtime.

-   **Thread Safety:** The class is **not thread-safe** for modification. However, because its state is established at load time and remains constant, it can be safely **read** by multiple threads simultaneously. For example, several game threads could process this interaction for different players without read-concurrency issues.

    **WARNING:** Any runtime modification of its fields is a severe anti-pattern and will lead to race conditions and unpredictable server behavior.

## API Surface
The primary public contract of this class is not a set of methods, but the configuration schema defined by its `CODEC`. Subclasses inherit this schema.

| Symbol (Configuration Key) | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| StatModifiers | Object2FloatMap | O(N) | Defines the map of stat names (e.g., "hytale:health") to their modification values. Subject to multiple validators. |
| ValueType | Enum | O(1) | Specifies whether `StatModifiers` are absolute values or percentages. Defaults to `Absolute`. |
| Behaviour | Enum | O(1) | Defines how the modifier is applied (e.g., Add, Set, Multiply). Defaults to `Add`. |
| Entity | Enum | O(1) | The target of the stat change, relative to the interaction's source (e.g., USER, TARGET). Defaults to `USER`. |

## Integration Patterns

### Standard Usage
Developers do not instantiate or call this class directly in Java code. Instead, they define its behavior declaratively within a server configuration file. The Interaction System later retrieves and executes this pre-configured instance.

A conceptual configuration for a concrete implementation might look like this:
```json
{
  "type": "hytale:change_stat_interaction",
  "StatModifiers": {
    "hytale:health": 25.0,
    "hytale:mana": -10.0
  },
  "ValueType": "Absolute",
  "Behaviour": "Add",
  "Entity": "USER"
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new ConcreteChangeStatInteraction()`. The object must be created via the codec system to ensure that validation and the critical `afterDecode` logic (which resolves stat names to IDs) are executed.
-   **Runtime State Modification:** Do not modify fields like `valueType` or the `entityStats` map after the object has been loaded. The system treats this object as static configuration data.
-   **Bypassing Validation:** Attempting to create an instance without satisfying the codec's validators (e.g., providing an empty `StatModifiers` map) will fail asset loading.

## Data Pipeline
The data for this class flows from a static configuration file on disk to a runtime effect on a game entity.

> Flow:
> Server Asset File (JSON) -> Hytale Codec Engine -> **ChangeStatBaseInteraction.CODEC** -> In-Memory Instance (with resolved stat IDs) -> Interaction System Execution -> EntityStatsModule applies changes to target entity

