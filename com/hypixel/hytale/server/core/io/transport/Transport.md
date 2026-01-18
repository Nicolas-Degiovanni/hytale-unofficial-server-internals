---
description: Architectural reference for the Transport interface, the core abstraction for server network listeners.
---

# Transport

**Package:** com.hypixel.hytale.server.core.io.transport
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface Transport {
   TransportType getType();

   ChannelFuture bind(InetSocketAddress var1) throws InterruptedException;

   void shutdown();
}
```

## Architecture & Concepts

The Transport interface is a foundational abstraction within the Hytale server's networking layer. It defines the essential contract for any component capable of establishing a network listening endpoint. This component acts as the entry point for all incoming network traffic, abstracting the underlying I/O mechanism (e.g., TCP, UDP, or local in-memory channels for testing).

By programming against this interface, the server core remains decoupled from the specific network technology, such as Netty. This is an application of the **Strategy Pattern**, where concrete implementations like NettyTcpTransport or LocalTransport provide the specific binding and I/O handling logic. The server's bootstrap process selects and injects the appropriate Transport implementation based on configuration.

Its primary responsibility is to bind to a specified network address and prepare the I/O pipeline for accepting new client connections. It does not handle application-level data itself; rather, it creates the initial Channel which contains the pipeline of handlers that will process incoming data.

## Lifecycle & Ownership

As an interface, Transport itself has no lifecycle. The following describes the lifecycle of a concrete **implementation** of this interface.

-   **Creation:** A concrete Transport object is instantiated by the primary server bootstrap or initialization service, typically ServerEntryPoint. The choice of which implementation to create is determined by server configuration files.
-   **Scope:** The Transport instance is a long-lived, session-scoped object. It persists for the entire duration of the server's runtime, from initial binding to final shutdown.
-   **Destruction:** The shutdown method must be explicitly invoked during the server's shutdown sequence. This is a critical step to release all underlying network resources, such as port bindings and I/O worker threads. Failure to call shutdown will result in resource leaks and may prevent the JVM from terminating cleanly.

## Internal State & Concurrency

-   **State:** The interface defines no state. However, all concrete implementations are inherently stateful. They manage complex resources including Netty EventLoopGroups, ServerBootstrap configurations, and the active ServerChannel after a successful bind.
-   **Thread Safety:** Implementations of this interface **must be thread-safe**. While the bind and shutdown methods are typically invoked from the main server thread during startup and shutdown, they orchestrate operations across multiple I/O worker threads. The underlying machinery, such as Netty's event loops, is designed for high concurrency.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getType() | TransportType | O(1) | Returns an enum identifying the underlying transport mechanism (e.g., TCP, RAKNET). |
| bind(InetSocketAddress) | ChannelFuture | Blocking/Async | Initiates the asynchronous process of binding to a network address. Throws InterruptedException if the binding thread is interrupted. Returns a Netty ChannelFuture to which listeners can be attached to handle success or failure. |
| shutdown() | void | Blocking | Triggers the graceful shutdown of the transport. This operation blocks until all associated resources, including I/O threads and open channels, are released. |

## Integration Patterns

### Standard Usage

The Transport is configured and managed by the server's main bootstrap logic. Application-level code rarely interacts with it directly after startup.

```java
// Example from a server bootstrap sequence
Transport transport = transportProvider.get(serverConfig.getTransportType());
InetSocketAddress address = new InetSocketAddress(serverConfig.getHost(), serverConfig.getPort());

try {
    ChannelFuture bindFuture = transport.bind(address);
    bindFuture.sync(); // Block until the port is successfully bound
    
    // Server is now listening...
    
} catch (InterruptedException e) {
    // Handle bind failure
    transport.shutdown();
}
```

### Anti-Patterns (Do NOT do this)

-   **Multiple Binds:** Do not invoke bind more than once on a single Transport instance. The behavior is undefined and will almost certainly result in an exception or resource conflict.
-   **Neglecting Shutdown:** Failure to call shutdown during server termination is a critical error. This will leak native resources, including file descriptors and threads, preventing a clean exit of the application.
-   **Ignoring the Future:** The bind method is asynchronous. Do not assume the server is listening immediately after the call returns. Always use the returned ChannelFuture to synchronize and verify the outcome of the bind operation.

## Data Pipeline

The Transport interface is the **originator** of the server-side data pipeline. It does not process data packets but is responsible for creating the channel through which all data will eventually flow.

> Flow:
> **Transport.bind()** → Creates ServerSocketChannel → Accepts Client SocketChannel → ChannelInitializer adds handlers → Channel Pipeline (Decoders → Game Logic Handlers) → Upstream Events

