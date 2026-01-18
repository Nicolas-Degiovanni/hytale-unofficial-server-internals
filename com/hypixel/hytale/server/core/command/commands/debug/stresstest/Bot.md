---
description: Architectural reference for Bot
---

# Bot

**Package:** com.hypixel.hytale.server.core.command.commands.debug.stresstest
**Type:** Transient

## Definition
```java
// Signature
public class Bot extends SimpleChannelInboundHandler<Packet> {
```

## Architecture & Concepts

The Bot class is a self-contained, simulated game client designed exclusively for server stress testing. Each instance represents a single player connection, complete with its own network channel, state, and a rudimentary AI for movement. Architecturally, it functions as a bridge between the low-level Netty networking layer and a simplified, independent game loop.

By extending Netty's SimpleChannelInboundHandler, a Bot instance injects itself directly into the network processing pipeline. This allows it to react to server-sent packets on Netty's I/O threads. However, its primary proactive logic—simulating player movement and sending updates—is driven by a dedicated, shared ScheduledExecutorService. This decoupling is critical: it prevents the simulation logic of thousands of bots from blocking the high-performance Netty I/O threads, enabling massive-scale testing.

All Bot instances share a static Netty EventLoopGroup and a static ScheduledExecutorService. This resource-pooling strategy is a deliberate design choice to minimize thread overhead and context switching, allowing the system to support a high number of concurrent bot connections efficiently.

### Lifecycle & Ownership
-   **Creation:** A Bot is instantiated directly by a managing system, such as the StressTestStartCommand. The constructor is a heavyweight operation that immediately initiates a synchronous network connection to the server and schedules the recurring `tick` task. Construction is not complete until a TCP connection is established.
-   **Scope:** An instance persists for the duration of its network session. Its lifecycle is intrinsically tied to the underlying Netty SocketChannel.
-   **Destruction:** Termination is triggered by one of three events: an explicit call to `shutdown`, a disconnect packet from the server (`channelInactive`), or a network-level exception (`exceptionCaught`). In all cases, the Bot cancels its scheduled tick task, closes the network channel, and removes itself from the central `StressTestStartCommand.BOTS` tracking collection.

## Internal State & Concurrency
-   **State:** The Bot class is highly stateful and mutable. It maintains the bot's entity ID, world position (`pos`), rotation, current destination, and a queue of pending ping packets. This state is continuously modified by both incoming network packets and the internal `tick` method.

-   **Thread Safety:** This class is **not thread-safe** and its concurrent model must be understood to be used correctly. State is accessed and mutated by two distinct thread pools:
    1.  **Netty Worker Threads:** The `channelRead0` method and other channel handlers are executed on the shared `WORKER_GROUP`. These threads update the bot's state in response to server packets (e.g., teleportation, entity updates).
    2.  **Bot Executor Threads:** The `tick` method is executed on the shared `EXECUTOR` thread pool. This thread reads and writes position and rotation data to calculate movement.

    **WARNING:** There is no explicit locking between these two thread contexts. The design implicitly relies on the assumption that state updates from the network are atomic enough or infrequent enough to not cause critical race conditions with the `tick` loop. Direct external modification of a Bot's state will lead to unpredictable behavior and concurrency violations.

## API Surface

The primary public contract consists of the constructor for creation and the `shutdown` method for termination. Other public methods are largely for internal use by the Netty framework or the scheduled executor.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Bot(name, config, tickStepNanos) | constructor | O(N) | Creates a new bot. Blocks until the network connection is established. Throws exceptions on connection failure. |
| shutdown() | void | O(1) | Initiates a graceful shutdown of the bot, closing the connection and canceling its tick task. This is an asynchronous operation. |
| tick(dt) | void | O(1) | Executes a single simulation step. Calculates movement and sends updates to the server. Intended for internal execution only. |

## Integration Patterns

### Standard Usage

A Bot should be created and managed by a higher-level controller responsible for the stress test. The controller maintains a reference to each bot, initiating the test by creating them and ending it by calling `shutdown` on each instance.

```java
// Example from a hypothetical test manager
List<Bot> activeBots = new ArrayList<>();
BotConfig config = loadBotConfig(); // Load spawn, speed, etc.

for (int i = 0; i < 1000; i++) {
    try {
        String botName = "StressBot-" + i;
        Bot bot = new Bot(botName, config, 50_000_000); // 20 ticks/sec
        activeBots.add(bot);
    } catch (Exception e) {
        // Handle connection failure for a single bot
        logger.log("Failed to create bot " + i, e);
    }
}

// ... later, to end the test
for (Bot bot : activeBots) {
    bot.shutdown();
}
```

### Anti-Patterns (Do NOT do this)
-   **State Manipulation:** Do not directly modify public fields like `pos` or `rotation` from an external thread. All state changes must be driven by the server (via packets) or the internal `tick` method.
-   **Reusing Instances:** A Bot instance is bound to a single network connection. Once `shutdown` is called or the connection is lost, the instance is defunct. Do not attempt to reuse it; create a new instance instead.
-   **Invoking Tick Manually:** Do not call the `tick` method directly. It is designed to be executed by the internal `ScheduledExecutorService` at a fixed rate. Manual invocation will disrupt the simulation timing and cause concurrency issues.

## Data Pipeline

The Bot processes two primary data flows: an inbound flow for reacting to the server and an outbound flow for simulating player action.

> **Inbound Flow (Server to Bot):**
> Network Packet -> Netty `PacketDecoder` -> **Bot.channelRead0** -> Internal State Update (e.g., position from a teleport packet)

> **Outbound Flow (Bot to Server):**
> **Bot.tick** (on Executor thread) -> Calculate new position -> `createMovementPacket` -> Netty `PacketEncoder` -> Network Packet

