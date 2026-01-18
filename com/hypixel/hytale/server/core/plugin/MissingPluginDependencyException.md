---
description: Architectural reference for MissingPluginDependencyException
---

# MissingPluginDependencyException

**Package:** com.hypixel.hytale.server.core.plugin
**Type:** Transient

## Definition
```java
// Signature
public class MissingPluginDependencyException extends RuntimeException {
```

## Architecture & Concepts
The MissingPluginDependencyException is a specialized, unchecked exception that signals a catastrophic and unrecoverable configuration error during the server's bootstrap phase. Its primary architectural role is to enforce the server's fail-fast startup policy.

By extending RuntimeException, the engine designers have made a deliberate choice: this is not an error that services should attempt to catch and recover from. Instead, it is a fatal condition that must halt the server's initialization process to prevent unpredictable behavior, such as NoClassDefFoundError, during runtime. This exception is a critical component of the plugin dependency resolution system, ensuring that the server only enters a running state with a valid and complete set of plugins.

### Lifecycle & Ownership
- **Creation:** Instantiated and thrown exclusively by the internal PluginLoader service when its dependency resolution algorithm detects that a plugin's required dependency is not present or failed to load.
- **Scope:** Ephemeral. The object's lifecycle is confined to the call stack from the point it is thrown to the point it is caught by the server's main bootstrap exception handler.
- **Destruction:** The object is eligible for garbage collection immediately after being caught and logged. It holds no persistent state and is not referenced after the initial handling.

## Internal State & Concurrency
- **State:** Immutable. The exception's state, consisting of a descriptive message, is set once upon construction via the parent RuntimeException constructor and cannot be modified thereafter.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. However, instances are typically confined to the single thread responsible for server initialization, making concurrent access a non-applicable concern.

## API Surface
The public contract is limited to its constructor, which is invoked internally by the plugin framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| MissingPluginDependencyException(message) | Constructor | O(1) | Constructs the exception. This is not intended for direct use by plugin developers. |

## Integration Patterns

### Standard Usage
This exception is not intended to be caught or handled by user-level code or plugins. The standard interaction is for the server's top-level entry point to catch it, log the detailed message, and terminate the process with a non-zero exit code.

```java
// Simplified example of the server's bootstrap error handling
try {
    pluginLoader.resolveAndLoadPlugins();
} catch (MissingPluginDependencyException e) {
    // This is the intended handling path
    logger.fatal("Server startup failed due to a missing plugin dependency.", e);
    System.exit(1);
}
```

### Anti-Patterns (Do NOT do this)
- **Catching and Swallowing:** Catching this exception within a plugin's lifecycle and attempting to continue execution is a severe anti-pattern. This will mask a critical configuration error and lead to instability and subsequent, more obscure runtime errors.

- **Generic Handling:** Do not catch a broad parent type like Exception or RuntimeException to handle this case. The server's bootstrap logic must be able to distinguish this specific fatal error from other potential startup issues.

## Data Pipeline
This class does not process data; it represents a terminal state in a control flow.

> Flow:
> Plugin Dependency Resolution -> **MissingPluginDependencyException Thrown** -> Bootstrap Exception Handler -> Logger -> Process Termination

