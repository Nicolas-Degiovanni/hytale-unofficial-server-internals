---
description: Architectural reference for HytaleServer
---

# HytaleServer

**Package:** com.hypixel.hytale.server.core
**Type:** Singleton

## Definition
```java
// Signature
public class HytaleServer {
```

## Architecture & Concepts
The HytaleServer class is the central nervous system and entry point for the entire server application. It functions as the application kernel, responsible for orchestrating the server's lifecycle from initial boot to final shutdown. It is not a component of the real-time game loop; rather, it owns and manages the lifecycles of all primary server subsystems, including the PluginManager, CommandManager, EventBus, and HytaleServerConfig.

Its core responsibilities are:
1.  **Initialization:** Establishes the foundational environment, including logging, configuration loading, error reporting (Sentry), and low-level utilities (Netty, Threading).
2.  **Service Management:** Acts as the root service locator, holding singleton instances of critical managers that other parts of the engine query for.
3.  **Lifecycle Orchestration:** Manages the strict, sequential phases of server startup (Setup, Asset Loading, Plugin Start) and shutdown.
4.  **State Tracking:** Maintains the global server state (Booting, Booted, Shutting Down) using thread-safe primitives to ensure consistent behavior across the application.

This class is the highest-level container for the server process. All other components operate within the context and lifecycle it defines.

### Lifecycle & Ownership
- **Creation:** The HytaleServer is instantiated exactly once by the main application entry point at process launch. The constructor immediately sets the static singleton instance and triggers the entire boot sequence.
- **Scope:** The instance is a true singleton that persists for the entire lifetime of the server process. A dedicated keep-alive thread holds a semaphore (`aliveLock`) to prevent premature JVM exit until the shutdown sequence explicitly releases it.
- **Destruction:** Destruction is a controlled, multi-stage process initiated by calling `shutdownServer`. This can be triggered by an internal command, a critical error, or an external OS signal (SIGINT) caught by a JVM shutdown hook. The `shutdown0` method orchestrates the graceful shutdown of all subsystems before releasing the `aliveLock` and terminating the process. A failsafe mechanism will force-halt the JVM if the graceful shutdown hangs.

## Internal State & Concurrency
- **State:** The HytaleServer is highly stateful and mutable. It holds direct references to all core server managers and tracks its current lifecycle phase (e.g., booting, booted, shutdown reason). It also manages the server configuration object, which is periodically saved to disk.
- **Thread Safety:** This class is designed for concurrent access. Its lifecycle state is managed by atomic primitives (`AtomicBoolean`, `AtomicReference`, `Semaphore`) to prevent race conditions during boot and shutdown transitions. While the HytaleServer instance itself is thread-safe, the managers it provides (like PluginManager) may have their own distinct threading models and safety guarantees. The boot and shutdown sequences are carefully designed to be idempotent; for example, calling `shutdownServer` multiple times will only trigger the shutdown process once.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | HytaleServer | O(1) | Static accessor for the singleton server instance. |
| getEventBus() | EventBus | O(1) | Returns the central event bus for system-wide communication. |
| getPluginManager() | PluginManager | O(1) | Returns the manager responsible for plugin lifecycle. |
| getCommandManager() | CommandManager | O(1) | Returns the manager for registering and dispatching commands. |
| getConfig() | HytaleServerConfig | O(1) | Returns the loaded server configuration object. |
| shutdownServer(reason) | void | O(N) | Initiates a graceful server shutdown. This is the only safe way to stop the server programmatically. |
| isBooted() | boolean | O(1) | Returns true if the server has completed its entire boot sequence and is fully operational. |
| isShuttingDown() | boolean | O(1) | Returns true if the server has begun its shutdown sequence. |

## Integration Patterns

### Standard Usage
The HytaleServer acts as a service locator. Code should never store a local reference to it but should instead retrieve the singleton instance on-demand to access core systems.

```java
// Correctly accessing a core system via the singleton
HytaleServer server = HytaleServer.get();
PluginManager pluginManager = server.getPluginManager();
pluginManager.start();
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new HytaleServer()`. The application's main entry point is solely responsible for its creation. A second instantiation would overwrite the static singleton, leading to catastrophic state corruption.
- **Premature Access:** Do not attempt to access managers like the PluginManager or Universe before the server has fully booted. Use the `isBooted()` flag or listen for the `BootEvent` on the event bus to ensure systems are ready.
- **Direct Shutdown Method:** Do not call `shutdown0`. This internal method bypasses the thread-safe state checks. Always use the public `shutdownServer` method to initiate a shutdown.

## Data Pipeline
The HytaleServer does not process a continuous stream of data. Instead, it orchestrates a **control flow** for the server's lifecycle.

> **Boot Control Flow:**
> Process Start -> **new HytaleServer()** -> Core Initializations (Logging, Config, Sentry) -> `boot()` method -> PluginManager Setup -> `LoadAssetEvent` Dispatch -> Asset Validation -> PluginManager Start -> Universe Initialization -> `BootEvent` Dispatch -> Server Ready

> **Shutdown Control Flow:**
> `shutdownServer()` call -> Shutdown Thread Spawned -> `shutdown0()` method -> `ShutdownEvent` Dispatch -> PluginManager Shutdown -> CommandManager Shutdown -> Final Config Save -> `aliveLock` Released -> Process Exit

