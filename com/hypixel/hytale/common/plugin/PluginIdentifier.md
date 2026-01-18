---
description: Architectural reference for PluginIdentifier
---

# PluginIdentifier

**Package:** com.hypixel.hytale.common.plugin
**Type:** Transient / Value Object

## Definition
```java
// Signature
public class PluginIdentifier {
```

## Architecture & Concepts
The PluginIdentifier is a fundamental **Value Object** that provides a canonical, unique identity for a plugin within the Hytale engine. It enforces a namespaced structure, `group:name`, which is critical for preventing naming collisions between plugins from different authors or modules. This pattern is analogous to coordinate systems used in dependency management tools like Maven or Gradle.

Its primary role is to act as a stable, serializable key. Systems like the PluginManager, dependency resolvers, and inter-plugin communication APIs rely on PluginIdentifier to look up, reference, and manage plugin instances and their metadata.

The class is designed to be **immutable**. Once an instance is created, its identity cannot be changed. This design choice is crucial for its use as a reliable key in collections like HashMaps and for safe sharing across concurrent systems without the need for synchronization.

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand whenever a plugin needs to be referenced. There are two primary creation pathways:
    1.  From a PluginManifest object during the initial plugin loading and discovery phase.
    2.  From a string representation via the static factory method `fromString`, typically when resolving dependencies or handling API calls that reference a plugin by its string name.
- **Scope:** PluginIdentifier instances are transient and short-lived. They are not managed by any container or registry. They exist only as long as they are referenced, for example as a local variable, a method parameter, or a key in a map.
- **Destruction:** Cleanup is managed entirely by the Java Garbage Collector. When an instance is no longer reachable, it is automatically reclaimed. No explicit destruction or cleanup methods exist or are required.

## Internal State & Concurrency
- **State:** **Immutable**. The internal `group` and `name` fields are final and are assigned only once during construction. The object's state cannot be modified after it has been created.
- **Thread Safety:** This class is **inherently thread-safe**. Due to its immutability, a single PluginIdentifier instance can be safely passed between, stored, and read by multiple threads simultaneously without any risk of data corruption or race conditions. No external locking is ever required when interacting with this object.

## API Surface
The public contract is focused on creation, access, and serialization. The correct implementation of `equals` and `hashCode` is vital to its function as a key.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PluginIdentifier(group, name) | Constructor | O(1) | Creates a new identifier from its constituent parts. |
| PluginIdentifier(manifest) | Constructor | O(1) | Convenience constructor to create an identifier from a manifest. |
| fromString(str) | static PluginIdentifier | O(N) | Parses a `group:name` string into an identifier. Throws IllegalArgumentException on malformed input. |
| getGroup() | String | O(1) | Returns the group component of the identifier. |
| getName() | String | O(1) | Returns the name component of the identifier. |
| toString() | String | O(N) | Serializes the identifier back to its `group:name` string representation. |
| equals(o) | boolean | O(N) | Performs a value-based equality check. |
| hashCode() | int | O(N) | Computes a hash code based on the value of the group and name. |

## Integration Patterns

### Standard Usage
The most common use case is creating an identifier from a string and using it to query a central plugin registry or manager.

```java
// A system needs to interact with the core engine plugin.
PluginIdentifier corePluginId = PluginIdentifier.fromString("hytale:engine_core");

// Use the identifier as a key to retrieve the loaded plugin's main instance.
// WARNING: The lookup may return null if the plugin is not loaded or does not exist.
Plugin corePlugin = pluginManager.getPlugin(corePluginId);

if (corePlugin != null) {
    corePlugin.doWork();
}
```

### Anti-Patterns (Do NOT do this)
- **Manual String Parsing:** Do not manually split the identifier string using `String.split(":")`. Always use the provided `PluginIdentifier.fromString` static factory method. This ensures that validation logic is consistently applied and prevents subtle bugs from malformed inputs.
- **Identity Comparison:** Never use the `==` operator to compare two PluginIdentifier instances. As this is a value object, multiple instances can exist that represent the same logical plugin. Always use the `.equals()` method for comparison.
- **Null Inputs:** The class contract, enforced by annotations, forbids null arguments in its constructors and factory method. Passing null will result in a runtime exception. Always validate inputs before creating an identifier.

## Data Pipeline
PluginIdentifier acts as a data key, not a data processor. Its primary role is to transition a plugin's identity from a configuration file format to a type-safe, in-memory representation used for lookups.

> Flow:
> `plugin.json` (on disk) -> Deserialized to `PluginManifest` -> **`PluginIdentifier`** (created from manifest) -> Used as key in `PluginRegistry` Map -> Plugin Lookup

---

