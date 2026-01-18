---
description: Architectural reference for ServerManager
---

# ServerManager

**Package:** com.hypixel.hytale.server.core.io
**Type:** Singleton

## Definition
```java
// Signature
public class ServerManager extends JavaPlugin {
```

## Architecture & Concepts
The ServerManager is a core server plugin responsible for managing the server's network lifecycle and abstracting the underlying transport protocol. It acts as the primary interface for creating, managing, and destroying the network sockets that listen for incoming client connections.

Its key architectural functions are:

*   **Transport Abstraction:** It decouples the server logic from the specific network transport implementation (e.g., TCP or QUIC). The desired transport is selected at startup via server options, and the ServerManager instantiates the corresponding Transport object.
*   **Listener Management:** It maintains the state of all active network listeners (represented by Netty Channel objects). It provides a centralized API to bind to new addresses and unbind existing listeners.
*   **Initialization Orchestration:** The startup process, including transport setup and initial socket binding, is managed asynchronously using CompletableFuture. This prevents the main server thread from blocking on network I/O during the boot sequence.
*   **Packet Handler Wiring:** It provides a mechanism to register factories for SubPacketHandlers. These handlers are later instantiated and attached to the primary GamePacketHandler, effectively composing the server's network protocol logic from various modules.

As a JavaPlugin, its lifecycle is managed by the server's plugin system, ensuring it is initialized at the correct stage of the server boot process and shut down gracefully.

## Lifecycle & Ownership
- **Creation:** The ServerManager instance is created by the Hytale server's plugin loader during the initial bootstrap phase. The constructor immediately assigns the new object to a static field, enforcing the singleton pattern.
- **Scope:** The instance is a session-scoped singleton. It persists for the entire lifetime of the server process, from startup to shutdown.
- **Destruction:** The shutdown method is invoked by the plugin system when the server receives a shutdown signal. This method orchestrates a graceful network shutdown by disconnecting all players, closing all listening sockets, and releasing resources held by the underlying transport layer.

The initialization is a multi-stage, asynchronous process:
1.  **Constructor:** The static instance is set.
2.  **init():** The selected Transport implementation (TCP or QUIC) is instantiated. This runs asynchronously, tracked by the registerFuture.
3.  **start():** This method waits for the transport initialization to complete and then proceeds to bind the initial set of listening sockets as defined by server options. This stage is tracked by the bootFuture.

**WARNING:** Critical server systems should call waitForBindComplete before attempting to interact with the network to ensure the listeners are active.

## Internal State & Concurrency
- **State:** The ServerManager is stateful and highly mutable. Its primary state consists of:
    - A list of active Netty Channel objects representing the listening sockets.
    - A reference to the active Transport implementation.
    - A list of functions for creating SubPacketHandlers.
    - CompletableFuture objects to track the asynchronous initialization state.

- **Thread Safety:** The class is designed to be accessed primarily from the main server thread, but provides guarantees for specific concurrent operations.
    - The internal list of listener channels, **listeners**, is a CopyOnWriteArrayList. This makes read operations (like getListeners) lock-free and safe from any thread. Write operations (binding or unbinding a listener) are expensive but infrequent, making this a suitable choice.
    - The **subPacketHandlers** list is an ObjectArrayList and is **not thread-safe**. It is assumed that all calls to registerSubPacketHandlers occur during the single-threaded server setup phase. Modifying this list from multiple threads after startup will lead to undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | ServerManager | O(1) | Provides static access to the singleton instance. |
| bind(address) | boolean | High (Network I/O) | Binds a new listener to the specified socket address. Blocks until the operation completes. |
| unbindAllListeners() | void | High (Network I/O) | Closes all active listeners. This is a core part of the shutdown sequence. |
| getPublicAddress() | InetSocketAddress | O(N) | Iterates through active listeners to find a suitable public-facing IP address. Throws SocketException. |
| waitForBindComplete() | void | O(1) | Blocks the calling thread until the asynchronous startup and binding process is complete. |
| registerSubPacketHandlers(supplier) | void | O(1) | Registers a factory function used to create and wire up a SubPacketHandler. Not thread-safe. |

## Integration Patterns

### Standard Usage
The ServerManager is a globally accessible service. Other plugins or core systems retrieve the singleton instance to perform network-related setup or queries.

```java
// During plugin setup, register a custom packet handler
ServerManager serverManager = ServerManager.get();
serverManager.registerSubPacketHandlers(MyCustomPacketHandler::new);

// In a separate system, wait for the network to be ready
serverManager.waitForBindComplete();
InetSocketAddress address = serverManager.getPublicAddress();
if (address != null) {
    // Register this server with a master server list
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ServerManager()`. The instance is created and managed exclusively by the server's plugin framework. Always use `ServerManager.get()` to retrieve the active instance.
- **Concurrent Handler Registration:** Do not call `registerSubPacketHandlers` from multiple threads or after the server has finished its initial setup phase. The underlying collection is not thread-safe and is not designed for modification during runtime.
- **Ignoring Startup Futures:** Do not attempt to get the server's public address or assume the server is listening for connections immediately after startup. Always call `waitForBindComplete` to ensure the asynchronous binding process has finished.

## Data Pipeline
The ServerManager is not directly involved in the per-packet data flow. Instead, it is the architect of the pipeline's entry point. It establishes the listening sockets and configures the initial set of handlers that will process incoming data.

> Flow:
> Client Connection Request -> **ServerManager (via Transport)** -> Netty Channel Created -> Netty Pipeline (Decoders, GamePacketHandler) -> **SubPacketHandlers (Registered via ServerManager)** -> Universe/Game Logic

---

