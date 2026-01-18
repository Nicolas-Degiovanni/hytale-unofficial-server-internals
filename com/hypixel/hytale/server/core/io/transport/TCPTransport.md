---
description: Architectural reference for TCPTransport
---

# TCPTransport

**Package:** com.hypixel.hytale.server.core.io.transport
**Type:** Transient Service

## Definition
```java
// Signature
public class TCPTransport implements Transport {
```

## Architecture & Concepts
The TCPTransport class is a concrete implementation of the Transport interface, providing the foundational TCP networking layer for the Hytale game server. It is built directly on the Netty high-performance asynchronous event-driven network framework.

This class's primary responsibility is to bootstrap the server's network listener. It configures and manages the two core Netty thread pools (EventLoopGroups):
*   **Boss Group:** A single-threaded pool responsible exclusively for accepting new incoming TCP connections from clients.
*   **Worker Group:** A multi-threaded pool that handles all I/O operations (reading, writing, processing) for every established client connection.

TCPTransport acts as the configuration and lifecycle authority for the server's Netty instance. It uses the NettyUtil helper to select the most efficient, platform-specific channel implementation (e.g., Epoll on Linux, NIO on other platforms). Crucially, it attaches the HytaleChannelInitializer to the pipeline, which is responsible for setting up the sequence of decoders, encoders, and game-logic handlers for each new client connection. This class does not handle game packets itself; it creates the environment where they can be processed.

### Lifecycle & Ownership
-   **Creation:** Instantiated directly via its constructor, typically by a higher-level server management class during the initial startup sequence. The constructor is a blocking operation, as it fully initializes and registers the Netty bootstrap configuration, which can throw an InterruptedException.
-   **Scope:** The object's lifecycle is bound to the server's own runtime. It is created once when the server starts and persists until the server begins its shutdown process.
-   **Destruction:** The shutdown method must be explicitly called to terminate the transport layer. This method gracefully shuts down the boss and worker thread pools, releasing all bound network ports and associated system resources. Failure to call this method will result in resource leaks, preventing a clean server exit.

## Internal State & Concurrency
-   **State:** This class is highly stateful. It holds direct references to the Netty EventLoopGroup and ServerBootstrap instances. These are not passive data objects; they represent active thread pools and network resource configurations. The state is established during construction and is considered immutable for the object's lifetime until shutdown is initiated.
-   **Thread Safety:** The public API of this class is **not thread-safe** and is designed to be controlled by a single thread (e.g., the main server thread). The constructor, bind, and shutdown methods should be called sequentially from a single point of control. However, the underlying Netty components it manages are the core of the server's concurrent I/O processing, safely handling thousands of client connections across the worker threads. This class is the gateway to that concurrent system, not a concurrent utility itself.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getType() | TransportType | O(1) | Returns the enum constant TransportType.TCP. |
| bind(address) | ChannelFuture | Blocking I/O | Binds the server to the specified socket address. This is a synchronous, blocking call that completes only when the port is successfully bound or an error occurs. |
| shutdown() | void | Blocking I/O | Initiates a graceful shutdown of the Netty event loops. This call blocks for up to one second per group to allow for clean termination. |

## Integration Patterns

### Standard Usage
The class is intended to be used by a central server controller. The controller instantiates the transport, binds it to a port, and holds the reference until the server is ready to shut down.

```java
// In a server startup method
try {
    TCPTransport transport = new TCPTransport();
    // The bind call will block until the server is listening
    transport.bind(new InetSocketAddress("0.0.0.0", 25565));
    // Store the transport instance for later shutdown
    this.serverTransport = transport;
} catch (InterruptedException e) {
    // Handle server startup failure
    Thread.currentThread().interrupt();
}

// In a server shutdown method
if (this.serverTransport != null) {
    this.serverTransport.shutdown();
}
```

### Anti-Patterns (Do NOT do this)
-   **Resource Leaking:** Never discard a TCPTransport instance without calling shutdown. This will leave the boss and worker thread pools running, preventing the JVM from exiting and holding onto the network port.
-   **Multiple Bind Calls:** The underlying bootstrap is not designed to be bound multiple times. A new instance should be created if a different configuration is required.
-   **Ignoring Exceptions:** The constructor and bind methods can throw InterruptedException. This indicates a critical failure in setting up the network layer and must be handled to prevent the server from starting in an invalid state.

## Data Pipeline
TCPTransport sits at the very beginning of the server's data processing pipeline. It accepts raw byte streams from the network and hands them off to the Netty channel pipeline, which is configured by HytaleChannelInitializer.

> Flow:
> Raw TCP Connection -> **TCPTransport (via Netty)** -> HytaleChannelInitializer -> [Byte-to-Packet Decoders] -> [Packet Handlers] -> Game Logic

