---
description: Architectural reference for PluginState
---

# PluginState

**Package:** com.hypixel.hytale.server.core.plugin
**Type:** Enumeration / Data Type

## Definition
```java
// Signature
public enum PluginState {
```

## Architecture & Concepts
The PluginState enum defines the discrete, sequential states of the server plugin lifecycle. It serves as the canonical model for the state machine managed by the PluginManager. This enumeration is not an active component; rather, it provides the vocabulary and constraints for tracking a plugin's progression from initial loading to final shutdown.

Its primary architectural role is to enforce a strict and predictable lifecycle, preventing plugins from executing logic in an invalid state (e.g., handling a game event before being fully enabled). Each state represents a well-defined phase, ensuring that initialization, operation, and cleanup occur in the correct order.

## Lifecycle & Ownership
As an enumeration, PluginState's lifecycle is managed directly by the Java Virtual Machine and is distinct from the lifecycle of the plugin objects it describes.

- **Creation:** The enum constants (NONE, SETUP, ENABLED, etc.) are instantiated by the JVM when the PluginState class is loaded. They are compile-time constants.
- **Scope:** These constants are static and exist for the entire lifetime of the server application. They are globally accessible, application-scoped singletons.
- **Destruction:** The constants are garbage collected only when the defining ClassLoader is unloaded, which typically occurs during a full server shutdown.

## Internal State & Concurrency
- **State:** Inherently immutable. The enum constants themselves have no mutable state. They are fixed references.
- **Thread Safety:** Guaranteed thread-safe by the Java Language Specification. Any thread can safely read and compare these enum constants without synchronization. The state of a *plugin object* that references a PluginState is not guaranteed to be thread-safe and must be managed by its owner, typically the PluginManager.

## API Surface
The API consists of the set of defined state constants.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| NONE | PluginState | N/A | The initial state of a plugin instance before the lifecycle has begun. |
| SETUP | PluginState | N/A | The plugin is being configured and its dependencies are being resolved. |
| START | PluginState | N/A | The plugin's primary initialization logic is being executed. |
| ENABLED | PluginState | N/A | The plugin is fully operational and integrated into the server loop. |
| SHUTDOWN | PluginState | N/A | The plugin is being gracefully terminated and is releasing resources. |
| DISABLED | PluginState | N/A | The final state. The plugin is fully unloaded and inactive. |

## Integration Patterns

### Standard Usage
PluginState should be used by lifecycle management systems to check or transition the status of a plugin. Direct comparison is the intended use case.

```java
// Correctly checking the state of a plugin
if (plugin.getState() == PluginState.ENABLED) {
    // Safely execute operational logic
    plugin.onGameTick(currentTick);
}
```

### Anti-Patterns (Do NOT do this)
- **Manual State Transition:** Do not set a plugin's state variable directly. State transitions must be exclusively managed by the core PluginManager to ensure lifecycle hooks are fired correctly. Bypassing the manager leads to a desynchronized and unstable system.
- **Ordinal Comparison:** Do not use the enum's ordinal value for comparisons (e.g., `if (state.ordinal() > PluginState.SETUP.ordinal())`). This practice is extremely brittle and will break if the enum order is ever changed. Always compare constants directly.

## Data Pipeline
PluginState does not process data. Instead, it represents the stages in the plugin lifecycle pipeline.

> State Transition Flow:
> **NONE** → **SETUP** → **START** → **ENABLED** → **SHUTDOWN** → **DISABLED**

