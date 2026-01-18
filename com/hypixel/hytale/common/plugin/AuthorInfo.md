---
description: Architectural reference for AuthorInfo
---

# AuthorInfo

**Package:** com.hypixel.hytale.common.plugin
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class AuthorInfo {
```

## Architecture & Concepts
The AuthorInfo class is a simple data structure whose sole purpose is to represent the metadata of a plugin author. It is not a service or a manager, but rather a passive data container.

Its primary architectural significance lies in its integration with Hytale's serialization framework. The static final field **CODEC** is a self-contained definition of how to serialize and deserialize an AuthorInfo instance from a data source, such as a JSON or HOCON configuration file. This pattern makes the data object self-describing and decouples the plugin loading system from the specific format of the author metadata.

This class is designed to be instantiated by the framework during the plugin loading phase and subsequently treated as a read-only object by the rest of the engine.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale **Codec** framework during the deserialization of plugin metadata files. The provided `BuilderCodec` uses the `AuthorInfo::new` constructor reference, populating the fields from keys such as "Name", "Email", and "Url" in the source data.
- **Scope:** The lifetime of an AuthorInfo instance is strictly bound to its parent container, typically a `PluginInfo` or `PluginMetadata` object. It persists in memory only as long as the corresponding plugin's metadata is required by the system.
- **Destruction:** The object is marked for garbage collection when its parent `PluginInfo` object is discarded. This typically occurs during a plugin unload cycle or a full server/client shutdown.

## Internal State & Concurrency
- **State:** The internal state is **mutable** via public setters. However, this is a design choice to facilitate population by the codec framework. After initial deserialization, an AuthorInfo instance should be treated as effectively immutable.
- **Thread Safety:** This class is **not thread-safe**. The fields are accessed via unsynchronized getters and setters. Concurrent writes will result in race conditions. It is architecturally guaranteed to be populated on a single thread during plugin initialization. Subsequent reads from any thread are safe, but any modification after initialization is a severe anti-pattern and will lead to undefined behavior.

## API Surface
The public API is limited to simple data accessors. Setters are present for codec use and should not be invoked by application logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getName() | String | O(1) | Returns the configured name of the author. May be null. |
| getEmail() | String | O(1) | Returns the configured email of the author. May be null. |
| getUrl() | String | O(1) | Returns the configured URL or website of the author. May be null. |

## Integration Patterns

### Standard Usage
AuthorInfo objects should never be created or managed directly. They are accessed from a higher-level object that represents a loaded plugin.

```java
// Correctly access AuthorInfo from a parent PluginInfo object
PluginInfo pluginInfo = pluginRegistry.getPlugin("my.plugin.id").getInfo();
AuthorInfo author = pluginInfo.getAuthor(); // Example method

if (author != null && author.getName() != null) {
    System.out.println("Plugin created by: " + author.getName());
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new AuthorInfo()`. The object's state is meaningless unless populated by the codec system from a valid plugin metadata source.
- **Post-Load Mutation:** Modifying an AuthorInfo object after it has been loaded is a critical error. The system expects this data to be constant for the lifetime of the plugin.

```java
// DANGEROUS: Do not modify state after initialization
AuthorInfo author = plugin.getInfo().getAuthor();
author.setName("A new author"); // This will cause system-wide inconsistencies.
```

## Data Pipeline
AuthorInfo is a terminal point in the plugin metadata loading pipeline. It does not process or forward data; it simply holds it for consumption by other systems.

> Flow:
> Plugin Metadata File (e.g., plugin.json) -> File Reader -> Data Parser -> Hytale Codec Framework -> **AuthorInfo Instance** -> PluginInfo Container -> Game Systems (e.g., UI, Logging)

