---
description: Architectural reference for PlayerPacketFilter
---

# PlayerPacketFilter

**Package:** com.hypixel.hytale.server.core.io.adapter
**Type:** Functional Interface / Contract

## Definition
```java
// Signature
public interface PlayerPacketFilter extends BiPredicate<PlayerRef, Packet> {
```

## Architecture & Concepts
The PlayerPacketFilter interface defines a fundamental contract within the server's network processing pipeline. It represents a single, stateless decision point for incoming network packets, operating on a per-player basis. By implementing this interface, developers can create rules to conditionally accept or reject packets before they reach the core game logic.

This component embodies the **Strategy Pattern**, allowing the server's packet handling behavior to be dynamically composed and altered. Filters are chained together to form a multi-stage validation gauntlet. A packet must pass through all registered filters to be considered valid for processing. This mechanism is critical for:

- **Security:** Dropping malformed or unexpected packets from clients.
- **Anti-Cheat:** Invalidating packets that represent impossible actions, such as moving too fast.
- **State Management:** Rejecting packets that are irrelevant to a player's current state (e.g., blocking combat packets when a player is in a non-combat zone).
- **Flow Control:** Preventing packet spam or denial-of-service vectors at the earliest possible stage.

It acts as a gatekeeper, positioned between the low-level packet deserialization layer and the high-level game event handlers.

## Lifecycle & Ownership
As a functional interface, PlayerPacketFilter itself has no lifecycle. The lifecycle pertains to its concrete implementations.

- **Creation:** Implementations are typically created as stateless singletons or as lambda expressions during server bootstrap or module initialization. They are then registered with a central network pipeline manager or packet processor.
- **Scope:** The lifetime of a filter implementation is tied to its registration. Global filters may persist for the entire server session. Module-specific filters, such as those for a particular minigame, will live only as long as that module is active.
- **Destruction:** Implementations are eligible for garbage collection once they are deregistered from the network pipeline and no other references exist.

## Internal State & Concurrency
- **State:** The contract is inherently stateless. Implementations of PlayerPacketFilter **must** be designed to be stateless. Storing mutable, per-invocation data within a filter is a severe anti-pattern that will lead to race conditions.
- **Thread Safety:** Implementations **must be thread-safe**. The server's network engine is highly concurrent and will invoke the same filter instance from multiple I/O threads simultaneously for different players. All filtering logic should be self-contained and free of side effects.

## API Surface
The interface inherits its primary method from the Java standard library BiPredicate.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(PlayerRef player, Packet packet) | boolean | O(N) | Evaluates the given packet against a rule for the specified player. Returns **true** to allow the packet to proceed, **false** to drop it. Complexity is implementation-dependent but should be as close to O(1) as possible to avoid blocking the network thread. |

## Integration Patterns

### Standard Usage
Filters are typically implemented as lambdas and registered with the appropriate network processing service. The logic should be a concise, side-effect-free predicate.

```java
// Example: Registering a filter to block movement packets for frozen players
PacketProcessor processor = server.getPacketProcessor();

PlayerPacketFilter freezeFilter = (player, packet) -> {
    if (packet instanceof PlayerMovePacket) {
        return !player.hasEffect(Effects.FROZEN);
    }
    return true; // Allow all other packets
};

processor.addFilter(freezeFilter);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not store per-player state inside a filter instance. This will break under concurrency. Use the provided PlayerRef to look up state from a thread-safe service if necessary.
- **Blocking Operations:** Never perform database queries, file I/O, or any long-running computation within a filter. Doing so will stall a primary network I/O thread, causing catastrophic server-wide latency.
- **Modifying State:** A filter's sole responsibility is to return true or false. It **must not** modify the state of the PlayerRef or the Packet. State modification is the responsibility of downstream packet handlers, after a packet has been fully validated.

## Data Pipeline
This interface is a key stage in the server's inbound data pipeline. It provides an early exit for invalid data, protecting more resource-intensive systems downstream.

> Flow:
> Network I/O Thread -> Packet Deserializer -> **PlayerPacketFilter Chain** -> Packet Handler Dispatcher -> Game Logic Execution

