---
description: Architectural reference for LANDiscoveryThread
---

# LANDiscoveryThread

**Package:** com.hypixel.hytale.builtin.landiscovery
**Type:** Transient Worker Thread

## Definition
```java
// Signature
class LANDiscoveryThread extends Thread {
```

## Architecture & Concepts
The LANDiscoveryThread is a dedicated, long-running background process responsible for making a Hytale server discoverable on a local area network (LAN). It operates independently of the main server game loop, listening for specific UDP broadcast packets and responding with server metadata.

This component functions as a low-level network listener that implements a simple, custom discovery protocol. By binding directly to a multicast socket on port 5510, it decouples the responsibility of network discovery from the core server logic, ensuring that discovery requests can be handled with minimal performance impact on the main game thread.

Its primary architectural role is to act as a passive announcer. It does not initiate connections; it only listens for a specific request header (**HYTALE\_DISCOVER\_REQUEST**) and replies with a formatted data packet (**HYTALE\_DISCOVER\_REPLY**) containing the server's public address, port, name, and current player counts.

## Lifecycle & Ownership
-   **Creation:** Instantiated and started exclusively by the LANDiscoveryPlugin, typically during the server's initialization phase. The package-private visibility of the class enforces this ownership model.
-   **Scope:** The thread is configured as a daemon (**setDaemon(true)**), meaning its lifecycle is tied to the parent application. It will run for the entire duration of the server session but will not prevent the JVM from shutting down.
-   **Destruction:** The thread's main loop is designed to terminate when the thread is interrupted. The owning LANDiscoveryPlugin is responsible for calling interrupt on this thread during the server shutdown sequence. A **finally** block ensures that the underlying MulticastSocket is always closed, releasing the network port.

## Internal State & Concurrency
-   **State:** The primary internal state is the mutable MulticastSocket instance. This socket is created within the run method and is used for all network I/O operations for the lifetime of the thread.
-   **Thread Safety:** This class is **not thread-safe** and is designed to be confined to its own execution context. All operations on the internal socket are performed sequentially within the run method's loop. The public getSocket method exposes the internal socket, which is a significant concurrency risk.

    **Warning:** External access and modification of the socket via getSocket from another thread will lead to race conditions and undefined behavior, such as a SocketException if the socket is closed prematurely.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| run() | void | O(∞) | The thread's main entry point. Enters an infinite loop to receive and process UDP packets. Blocks on socket.receive(). |
| getSocket() | MulticastSocket | O(1) | Returns the internal MulticastSocket. **Warning:** Exposes internal mutable state. Avoid using this method. |

## Integration Patterns

### Standard Usage
This class is an internal component of the LAN discovery system and is not intended for direct use by developers. The system is enabled or disabled via server configuration, which controls whether the LANDiscoveryPlugin instantiates and starts this thread.

The correct interaction is with the managing plugin, not the thread itself.

```java
// Correct interaction is managed by the plugin, not direct instantiation.
// The following is a conceptual example of how the plugin would use it.

LANDiscoveryThread discoveryThread = new LANDiscoveryThread();
discoveryThread.start(); // The thread now runs in the background.

// ... later, during server shutdown
discoveryThread.interrupt(); // Signals the thread to terminate gracefully.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not create an instance of this class directly. The lifecycle is managed by the LANDiscoveryPlugin.
-   **Calling run() directly:** Never call the run method. This would execute the logic on the current thread, blocking it indefinitely. Always use the start method to begin execution in a new thread.
-   **External Socket Manipulation:** Do not use getSocket to access and modify the underlying socket from another thread. This breaks thread confinement and will cause severe network issues or crashes.

## Data Pipeline
The data flow is a simple request-response cycle over UDP. The thread acts as the central processor for this pipeline.

> Flow:
> Network UDP Datagram (Request) -> MulticastSocket -> **LANDiscoveryThread** -> ServerManager/Universe (State Fetch) -> Netty ByteBuf (Reply Assembly) -> MulticastSocket -> Network UDP Datagram (Reply)

