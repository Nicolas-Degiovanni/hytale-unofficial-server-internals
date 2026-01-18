---
description: Architectural reference for PluginManifest
---

# PluginManifest

**Package:** com.hypixel.hytale.common.plugin
**Type:** Transient Data Model

## Definition
```java
// Signature
public class PluginManifest {
```

## Architecture & Concepts

The PluginManifest class is a passive data structure that serves as the canonical blueprint for a Hytale plugin. It is the in-memory representation of a plugin's metadata, typically deserialized from a configuration file bundled with the plugin assets. This class is the primary contract between a plugin developer and the engine's Plugin Loader system.

Architecturally, its most significant feature is the static CODEC field. This field leverages Hytale's powerful Codec system to define a declarative, format-agnostic contract for serialization and deserialization. The engine can load a manifest from any data source (e.g., JSON, binary) that the Codec system supports, without altering the PluginManifest class itself.

The Plugin Loader consumes PluginManifest instances to perform critical startup tasks:
1.  **Dependency Resolution:** It reads the dependencies, optionalDependencies, and loadBefore maps to construct a directed acyclic graph (DAG) of plugins.
2.  **Load Order Calculation:** It performs a topological sort on the dependency graph to determine the correct initialization order, ensuring plugins are loaded only after their dependencies are met.
3.  **Plugin Identification:** It uses the group, name, and version fields to uniquely identify plugins and resolve version conflicts.

The class also provides a fluent `CoreBuilder` for programmatically defining manifests for internal, core-engine plugins that do not have external configuration files.

## Lifecycle & Ownership

-   **Creation:** A PluginManifest instance is never created directly by application code. Its instantiation is managed exclusively by two system-level mechanisms:
    1.  **Deserialization:** The primary path. The Plugin Loader reads a plugin's configuration file and uses the static `PluginManifest.CODEC` to deserialize the data stream into a new PluginManifest object.
    2.  **Core Builder:** Internal engine components are registered as plugins using the static `corePlugin` factory method, which provides a `CoreBuilder` to construct a manifest programmatically. This is reserved for bootstrapping the engine itself.

-   **Scope:** An instance of PluginManifest has a lifetime equivalent to its corresponding loaded plugin. It is created during the server or client startup sequence, held in memory by the Plugin Loader or a PluginContainer, and persists for the entire application session. It is treated as a read-only data source after the initial loading and dependency resolution phase.

-   **Destruction:** The object has no explicit destruction or cleanup logic. It is eligible for garbage collection when the application shuts down and the plugin registry is cleared.

## Internal State & Concurrency

-   **State:** The internal state is **highly mutable** during its initial construction phase. The Codec system or the CoreBuilder directly populates its fields. However, after being processed and registered by the Plugin Loader, it is intended to be treated as **effectively immutable**. Public getters for collections enforce this by returning unmodifiable views (e.g., `Collections.unmodifiableMap`).

-   **Thread Safety:** This class is **not thread-safe**. All fields are accessed and mutated without any synchronization primitives.

    **WARNING:** Accessing a PluginManifest instance from multiple threads is only safe under the strict condition that all mutations have been completed and its state is "frozen". This condition is naturally met by the single-threaded plugin loading process. Any attempt to modify a manifest via its setters or the `injectDependency` method after it has been published to the rest of the engine will lead to race conditions and undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDependencies() | Map | O(1) | Returns an unmodifiable map of required plugins and their required version ranges. Critical for dependency resolution. |
| getOptionalDependencies() | Map | O(1) | Returns an unmodifiable map of optional plugins. The loader will attempt to load these but will not fail if they are missing. |
| getLoadBefore() | Map | O(1) | Returns an unmodifiable map of plugins that this plugin must be loaded *before*. This provides fine-grained control over the load order. |
| inherit(PluginManifest) | void | O(N) | Populates fields in this manifest from a parent manifest if they are not already set. Used to reduce configuration boilerplate for sub-plugins. |
| corePlugin(Class) | CoreBuilder | O(1) | Static factory method. The sole entry point for programmatically defining a manifest for a core engine plugin. |
| injectDependency(...) | void | O(1) | **System-Use Only.** Mutates the dependency map. This is used internally by the loader to add implicit dependencies. |

## Integration Patterns

### Standard Usage

Developers do not typically interact with this class directly. The primary interaction is defining the corresponding configuration file. System code or other plugins may query the manifest of a loaded plugin to inspect its metadata.

```java
// Example of a system querying a loaded plugin's manifest
PluginRegistry registry = context.getService(PluginRegistry.class);
PluginContainer myPluginContainer = registry.getPlugin("my-plugin-name");

if (myPluginContainer != null) {
    PluginManifest manifest = myPluginContainer.getManifest();
    Semver version = manifest.getVersion();
    System.out.println("Running with My Plugin version: " + version);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new PluginManifest()`. The object will be incomplete and will cause `NullPointerException`s in the Plugin Loader. Manifests must be created via the Codec system or the CoreBuilder.
-   **Post-Load Mutation:** Do not call setters (e.g., `setName`, `setVersion`) or `injectDependency` on a manifest instance retrieved from a loaded `PluginContainer`. The Plugin Loader's dependency graph will become stale, leading to unpredictable load order and runtime errors.
-   **Relying on Mutable Collections:** Do not attempt to cast the collections returned by getters to their mutable counterparts. They are intentionally unmodifiable to guarantee state integrity after the loading phase.

## Data Pipeline

The PluginManifest is a key component in the plugin loading data pipeline. It acts as the structured, in-memory representation of raw configuration data.

> Flow:
> Plugin Configuration File -> I/O Byte Stream -> Hytale Codec Engine -> **PluginManifest Instance** -> Plugin Loader (for Dependency Graph Resolution) -> PluginContainer (stores manifest for runtime inspection)

