---
description: Architectural reference for ItemTranslationProperties
---

# ItemTranslationProperties

**Package:** com.hypixel.hytale.server.core.asset.type.item.config
**Type:** Transient

## Definition
```java
// Signature
public class ItemTranslationProperties implements NetworkSerializable<com.hypixel.hytale.protocol.ItemTranslationProperties> {
```

## Architecture & Concepts
ItemTranslationProperties is a server-side data model that encapsulates the internationalization (i18n) keys for a game item. It is not a service or manager, but rather a simple data container that represents a fragment of an item's static configuration.

Its primary architectural significance lies in its integration with the engine's data-driven asset system via the static **CODEC** field. This `BuilderCodec` defines the schema for how this object is deserialized from on-disk configuration files (e.g., JSON). The asset loading system uses this codec to parse the `translation` block within an item's definition and instantiate this class.

The implementation of the NetworkSerializable interface marks this class as part of the data contract between the server and client. It provides a standardized mechanism, `toPacket`, for converting this server-side configuration into a lightweight, network-optimized representation for transmission.

The metadata within the codec, specifically `UIEditor.LocalizationKeyField`, reveals its role in the Hytale content creation toolchain. This metadata instructs the editor on how to present these fields, providing auto-generated key suggestions to content creators, which enforces naming conventions and reduces errors.

## Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the Hytale asset pipeline during server startup. The `BuilderCodec` is invoked by the AssetLoader to deserialize an item's configuration file from disk into this in-memory object. Manual instantiation via `new` is rare and generally discouraged.
- **Scope:** The lifetime of an ItemTranslationProperties instance is bound to its parent item asset. It persists in memory as long as the item definition is loaded on the server. It is not a global object; a unique instance exists for each item asset that defines translation properties.
- **Destruction:** The object is marked for garbage collection when its parent item asset is unloaded from memory, typically during a server shutdown or a dynamic asset reload. It has no explicit destruction or cleanup logic.

## Internal State & Concurrency
- **State:** This class holds a mutable state consisting of two nullable String references: `name` and `description`. It is a simple Plain Old Java Object (POJO) and does not contain any caches, collections, or complex internal machinery.
- **Thread Safety:** **This class is not thread-safe.** It provides no internal synchronization. It is designed to be populated once during the server's single-threaded asset loading phase and treated as immutable thereafter.

**WARNING:** Modifying the state of this object from any thread after the initial asset loading phase is complete is an anti-pattern and will lead to unpredictable behavior and race conditions. Data within this class should be considered read-only at runtime.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getName() | String | O(1) | Returns the translation key for the item's name. May be null. |
| getDescription() | String | O(1) | Returns the translation key for the item's description. May be null. |
| toPacket() | com.hypixel.hytale.protocol.ItemTranslationProperties | O(1) | Creates and returns a network-safe packet representation of this object. |

## Integration Patterns

### Standard Usage
This object is not meant to be interacted with directly. It is a component of a larger item asset configuration. A developer's interaction is typically read-only, accessing it through the parent asset to retrieve localization keys.

```java
// Assume 'itemAsset' is a fully loaded item configuration object
ItemTranslationProperties translations = itemAsset.getTranslationProperties();

if (translations != null) {
    String nameKey = translations.getName();
    // Use the nameKey with the localization system to get the display name
}
```

### Anti-Patterns (Do NOT do this)
- **Runtime Modification:** Never modify the state of this object after server initialization. This configuration is considered static and may be read by multiple threads without locks.
- **Direct Instantiation:** Avoid using `new ItemTranslationProperties()`. This bypasses the asset pipeline and creates a disconnected object. This is only acceptable in isolated unit tests or highly specialized dynamic item generation systems.

## Data Pipeline
ItemTranslationProperties acts as a bridge between on-disk configuration and the server's runtime representation, which is then serialized for the network.

> Flow:
> Item Asset File (on disk) -> Asset Loading Service -> **ItemTranslationProperties.CODEC** -> In-Memory **ItemTranslationProperties** instance -> `toPacket()` method -> Network Packet -> Client Rendering

---

