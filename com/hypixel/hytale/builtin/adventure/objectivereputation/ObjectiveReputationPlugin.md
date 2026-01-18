---
description: Architectural reference for ObjectiveReputationPlugin
---

# ObjectiveReputationPlugin

**Package:** com.hypixel.hytale.builtin.adventure.objectivereputation
**Type:** Singleton

## Definition
```java
// Signature
public class ObjectiveReputationPlugin extends JavaPlugin {
```

## Architecture & Concepts
The ObjectiveReputationPlugin serves as a critical bridge component, integrating the Reputation system with the server's core Objective system. As a subclass of JavaPlugin, it is automatically discovered and loaded by the server's plugin manager during the bootstrap sequence.

Its sole architectural purpose is to perform one-time registrations and dependency configurations. It does not contain any business logic itself; rather, it injects the necessary components and configurations into other, larger systems. Specifically, it registers custom data codecs, objective completion handlers, and asset loading dependencies. This ensures that reputation-based objectives are correctly serialized, processed, and that their required assets are loaded in the proper order.

This class is a foundational piece for adventure mode functionality, enabling designers to create objectives that grant or require reputation with in-game factions.

### Lifecycle & Ownership
-   **Creation:** Instantiated exclusively by the server's plugin loading framework during startup. The framework provides a JavaPluginInit context object required by the constructor.
-   **Scope:** The plugin is designed as a singleton. Once its setup method completes, the static instance field is populated and persists for the entire lifetime of the server session.
-   **Destruction:** The object is destroyed and garbage collected when the server shuts down or the plugin is unloaded by the plugin manager.

## Internal State & Concurrency
-   **State:** The class holds a single piece of mutable state: the static *instance* field. This field is written to exactly once during the setup lifecycle method. After this initial write, it is effectively immutable for the remainder of the server's uptime.
-   **Thread Safety:** This class is **not thread-safe** during initialization. The assignment to the static *instance* field is not synchronized. However, the server's plugin loading process is designed to be single-threaded, mitigating this risk. Accessing the static *get* method from any thread is safe *after* the plugin's setup phase is complete. Calling *get* before initialization is complete will result in a NullPointerException.

## API Surface
The public API is minimal, exposing only the static accessor for the singleton instance. All other significant operations occur within the protected lifecycle method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | static ObjectiveReputationPlugin | O(1) | Retrieves the singleton instance of the plugin. Throws NullPointerException if called before the plugin is initialized. |

## Integration Patterns

### Standard Usage
This plugin is not intended for direct interaction in game logic. It is loaded automatically by the server. Other plugins that need to establish a hard dependency or interact with it would retrieve the instance via the static get method after the initial plugin loading phase.

```java
// Example from another plugin's setup method
// Ensures this plugin is loaded after the ObjectiveReputationPlugin
public void setup() {
    ObjectiveReputationPlugin reputationPlugin = ObjectiveReputationPlugin.get();
    if (reputationPlugin == null) {
        throw new IllegalStateException("ObjectiveReputationPlugin not loaded!");
    }
    // ... logic that depends on the reputation plugin being initialized
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new ObjectiveReputationPlugin()`. The plugin framework is the sole owner of this component's lifecycle. Attempting to create it manually will fail or lead to a detached, non-functional instance.
-   **Premature Access:** Do not call `ObjectiveReputationPlugin.get()` from a plugin that may load *before* this one. This will result in a NullPointerException. Declare dependencies correctly in your plugin's metadata to control load order.
-   **Manual Lifecycle Calls:** Never invoke the `setup` method directly. It is a protected lifecycle method managed exclusively by the plugin loader.

## Data Pipeline
This plugin does not participate in a continuous data pipeline. Instead, it acts as a one-time configuration agent that modifies the behavior of other major systems. Its influence is best understood as injecting logic into three distinct pipelines at server startup.

> **1. History Data Serialization Pipeline:**
> Server Shutdown -> ObjectiveRewardHistoryData Serialization -> **CODEC Registration** -> Disk
>
> *The plugin registers the ReputationObjectiveRewardHistoryData codec, enabling the server to correctly save and load player progress for reputation-based objectives.*

> **2. Objective Completion Pipeline:**
> Player Action -> Objective Completion Check -> **Completion Handler Registration** -> ReputationCompletion Logic -> Reward Grant
>
> *The plugin registers the ReputationCompletion handler, allowing the ObjectivePlugin to delegate the logic for completing reputation-based objectives.*

> **3. Asset Loading Pipeline:**
> Server Startup -> AssetRegistry -> **Dependency Injection** -> ReputationGroup Assets Loaded -> Objective Assets Loaded
>
> *The plugin injects a loading dependency, forcing the AssetRegistry to fully load all ReputationGroup assets before it begins loading ObjectiveAsset definitions. This prevents race conditions where an objective might reference a reputation group that has not yet been loaded into memory.*

