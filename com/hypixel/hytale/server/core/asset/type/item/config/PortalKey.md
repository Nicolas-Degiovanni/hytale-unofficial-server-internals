---
description: Architectural reference for PortalKey
---

# PortalKey

**Package:** com.hypixel.hytale.server.core.asset.type.item.config
**Type:** Configuration Model

## Definition
```java
// Signature
public class PortalKey {
```

## Architecture & Concepts
The PortalKey class is a pure data model that represents the configuration for an item component that can open a specific type of portal. It is not a service or a manager, but rather a passive data structure whose schema and validation rules are defined by its static **CODEC** field.

Architecturally, this class is a prime example of Hytale's **data-driven design**. Its primary role is to be deserialized from an asset configuration file (e.g., a JSON file defining an item) by the server's asset loading system. The `BuilderCodec` instance defines the contract for this deserialization, including expected field names ("PortalType", "TimeLimitSeconds"), data types, and validation logic.

Crucially, the codec integrates with the `PortalType` asset system via `PortalType.VALIDATOR_CACHE`. This ensures that a PortalKey cannot be loaded with an invalid `portalTypeId`, preventing runtime errors and guaranteeing data integrity at engine startup. The class acts as a strongly-typed, validated bridge between raw configuration data on disk and the in-memory representation used by game logic.

### Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the Hytale Codec framework during the server's asset loading phase. Game logic or plugins should never instantiate this class directly. The `AssetManager` invokes the static `CODEC` to construct and populate a PortalKey instance when it encounters the corresponding data block within an item's configuration file.
-   **Scope:** An instance of PortalKey has its lifetime bound to the parent asset that contains it, typically an item configuration. It is effectively an immutable component of that parent asset.
-   **Destruction:** The object is marked for garbage collection when its parent asset is unloaded from memory. There is no manual destruction or cleanup required.

## Internal State & Concurrency
-   **State:** Immutable. The internal fields `portalTypeId` and `timeLimitSeconds` are private and are only populated once during deserialization via the `CODEC`. There are no public setters, and the state of an instance cannot be changed after it is created.
-   **Thread Safety:** This class is inherently thread-safe. Due to its immutability, a loaded PortalKey instance can be safely read by multiple systems across different threads (e.g., the main game loop, AI threads, network threads) without requiring any locks or synchronization primitives.

## API Surface
The public API is minimal, designed for read-only access to the configuration data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPortalTypeId() | String | O(1) | Retrieves the unique identifier of the PortalType asset this key is associated with. |
| getTimeLimitSeconds() | int | O(1) | Retrieves the duration in seconds a portal opened by this key should remain active. Returns -1 if no limit is defined. |

## Integration Patterns

### Standard Usage
A game system, such as an item interaction handler, retrieves the PortalKey component from an item's configuration to execute game logic.

```java
// Assume 'itemConfig' is the fully loaded asset configuration for an item
// and 'world' is the game world instance.

PortalKey keyComponent = itemConfig.getComponent(PortalKey.class);

if (keyComponent != null) {
    String portalId = keyComponent.getPortalTypeId();
    int timeLimit = keyComponent.getTimeLimitSeconds();

    // Use the retrieved data to interact with other engine systems
    world.getPortalManager().openPortal(portalId, timeLimit);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new PortalKey()`. The resulting object would be uninitialized and invalid, as it bypasses the critical deserialization and validation logic handled by the `CODEC`. Always retrieve it as a component from a loaded asset.
-   **State Modification:** Do not attempt to modify the internal state of a PortalKey instance using reflection. The engine's systems rely on the immutability of these configuration objects for thread safety and predictable behavior.

## Data Pipeline
The PortalKey class is the terminal point of a configuration loading pipeline. Data flows from a raw text file on disk into a validated, type-safe, in-memory object.

> Flow:
> Item Asset File (e.g., `my_key.json`) -> Server Asset Loader -> Hytale Codec System -> **PortalKey Instance** -> Item Interaction System

