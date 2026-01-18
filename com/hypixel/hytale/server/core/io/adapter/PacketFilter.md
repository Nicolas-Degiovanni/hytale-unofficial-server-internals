---
description: Architectural reference for PacketFilter
---

# PacketFilter

**Package:** com.hypixel.hytale.server.core.io.adapter
**Type:** Contract / Functional Interface

## Definition
```java
// Signature
public interface PacketFilter extends BiPredicate<PacketHandler, Packet> {
```

## Architecture & Concepts
The PacketFilter interface defines a behavioral contract for intercepting and conditionally processing network packets. It is a core component of the server's network processing pipeline, acting as a predicate-based gatekeeper that determines whether an inbound packet should proceed to its designated PacketHandler.

By extending the standard Java functional interface BiPredicate, PacketFilter is designed to be implemented as a lightweight, stateless function. This design promotes a highly decoupled and composable network architecture. Filters can be chained, added, or removed dynamically to modify the packet processing flow without altering the core PacketHandler logic.

Its primary role is to enforce pre-conditions before a packet is handled. Common use cases include:
- **Security:** Dropping malformed or unexpected packets from a client.
- **State Management:** Allowing certain packets only when a client is in a specific state (e.g., authenticated, in-game, in-lobby).
- **Flow Control:** Throttling or rate-limiting specific packet types.

## Lifecycle & Ownership
As an interface, PacketFilter itself has no lifecycle. The lifecycle described here pertains to its concrete implementations.

- **Creation:** Implementations are typically instantiated as lambdas or anonymous classes during the setup of a network connection's processing pipeline (e.g., when a player joins the server). They are not meant to be managed by a dependency injection framework but are rather configured directly by the network management layer.
- **Scope:** The lifetime of a PacketFilter implementation is tied to the scope of the pipeline or handler it is registered with. For instance, a filter added to a player's connection pipeline will exist only for the duration of that player's session.
- **Destruction:** An implementation is eligible for garbage collection when the owning component, such as the network connection or channel pipeline, is torn down and dereferenced.

## Internal State & Concurrency
- **State:** The PacketFilter contract is inherently stateless. Implementations **must** be designed as pure functions where possible, with their output depending solely on their inputs (the PacketHandler and the Packet).
    - **WARNING:** Introducing mutable state into a filter implementation is a significant anti-pattern. Stateful filters can create subtle race conditions and make the network pipeline's behavior difficult to predict, especially in a multi-threaded server environment.

- **Thread Safety:** The test method is guaranteed to be invoked on a network I/O thread. Therefore, all implementations **must be thread-safe**. If state is unavoidable, all access to that state must be protected by appropriate concurrency controls (e.g., locks, atomic variables). The high-performance nature of the network layer demands that any such synchronization be non-blocking or extremely short-lived.

## API Surface
The public contract consists of a single method inherited from BiPredicate.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(PacketHandler, Packet) | boolean | O(N) | Executes the filter logic. Returns **true** to permit the packet to continue down the processing chain, or **false** to discard it. The complexity N is determined by the implementation and should be kept as close to O(1) as possible. |

## Integration Patterns

### Standard Usage
PacketFilter is intended to be used as a lambda expression or method reference, which is then registered with a higher-level network pipeline or dispatcher component. This allows for concise and declarative filtering rules.

```java
// A hypothetical connection manager adding a filter to a player's pipeline.
// This filter ensures that only authenticated players can send ChatPackets.

PlayerConnectionPipeline pipeline = player.getPipeline();

PacketFilter chatFilter = (handler, packet) -> {
    if (packet instanceof ChatPacket) {
        // The decision is based on the state of the handler's owner
        return handler.getPlayer().isAuthenticated();
    }
    // Allow all other packet types to pass through
    return true;
};

pipeline.addFilter(chatFilter);
```

### Anti-Patterns (Do NOT do this)
- **Blocking Operations:** Never perform file I/O, database queries, or any other blocking operations within the test method. Doing so will stall the network I/O thread, catastrophically impacting server throughput and responsiveness.
- **Stateful Implementations:** Avoid creating filters that maintain their own mutable state. This can lead to concurrency issues and non-deterministic behavior. Use the state available on the PacketHandler or Packet instead.
- **Complex Business Logic:** Filters should be simple predicates. Complex, multi-step business logic belongs inside a dedicated PacketHandler, not the filter. The filter's job is to say "yes" or "no" quickly.

## Data Pipeline
PacketFilter is positioned early in the server's inbound data pipeline, immediately after a raw byte stream has been decoded into a structured Packet object but before it is dispatched to the main game logic.

> Flow:
> Network Byte Stream -> Packet Decoder -> **PacketFilter** -> PacketHandler -> Game Logic Execution

