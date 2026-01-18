---
description: Architectural reference for NetworkCommand
---

# NetworkCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility.net
**Type:** Configuration

## Definition
```java
// Signature
public class NetworkCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The NetworkCommand class is a command collection that serves as a centralized, developer-facing control panel for manipulating server-side network behavior and related physics systems. It is not part of the core gameplay loop but rather a utility for debugging, testing, and performance tuning.

This class acts as a high-level entry point, delegating specific actions to its nested sub-commands. These sub-commands directly interface with low-level systems, such as the Netty channel pipeline for latency simulation or global static flags that control physics prediction algorithms. Its primary architectural role is to expose these internal toggles and mechanisms to server administrators and developers through the standard command system, abstracting away the complexity of direct system manipulation.

The command is composed of three distinct functional areas:
1.  **Latency Simulation:** Modifies a specific player's network channel to introduce artificial delay, simulating high-latency conditions.
2.  **Server Knockback Prediction:** Toggles a global server-side flag that controls whether the server speculatively applies knockback physics.
3.  **Client Knockback Debugging:** Toggles a global flag that instructs clients to render debug information for knockback calculations.

## Lifecycle & Ownership
-   **Creation:** A single instance of NetworkCommand is created by the server's command registration system during the initial server bootstrap phase. It discovers this class and its sub-commands, adding them to a central command registry.
-   **Scope:** The object is a stateless container whose instance persists for the entire duration of the server session. It is not created on a per-player or per-command basis.
-   **Destruction:** The instance is dereferenced and eligible for garbage collection only when the server is shutting down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** The NetworkCommand object and its sub-command objects are themselves stateless. However, their purpose is to mutate highly sensitive, shared global state.
    -   The LatencySimulation command modifies the state of a player's specific Netty Channel pipeline via the LatencySimulationHandler. This state is external and managed on a per-connection basis.
    -   The ServerKnockback and DebugKnockback commands modify global, static boolean flags within the KnockbackSystems and KnockbackPredictionSystems classes, respectively. This constitutes a modification of shared, mutable global state.

-   **Thread Safety:** This class is not thread-safe by design. The command system guarantees that all `execute` and `executeSync` methods are invoked exclusively on the main server thread. This serialization ensures that modifications to global state (like the knockback flags) and per-player channel pipelines occur atomically with respect to the game loop, preventing race conditions.

    **Warning:** Any attempt to invoke these command handlers from an asynchronous context or a worker thread will lead to state corruption and severe server instability.

## API Surface
The primary API is not programmatic but rather the command structure exposed to a user via the server console or in-game chat.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| network latencysimulation set | Command | O(1) | Targets a player and injects a latency simulation handler into their network pipeline with a specified delay. |
| network latencysimulation reset | Command | O(1) | Targets a player and removes any active latency simulation from their network pipeline. |
| network serverknockback | Command | O(1) | Toggles the global boolean flag for server-side knockback prediction. |
| network debugknockback | Command | O(1) | Toggles the global boolean flag that enables client-side knockback position debugging. |

## Integration Patterns

### Standard Usage
These commands are intended to be executed by a server administrator or developer through the console or in-game chat.

```java
// Example: Simulating 150ms of latency for a player named "dev"
/network latencysimulation set dev 150

// Example: Toggling the server-side knockback prediction system
/net serverknockback
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not create an instance using `new NetworkCommand()`. The command system handles instantiation and registration. A manually created instance will not be registered and will have no effect.
-   **Programmatic Invocation:** Do not attempt to get an instance of this command and call its `execute` methods directly. This bypasses the entire command processing pipeline, including argument parsing, permission checks, and context setup, which is unsafe and will likely result in a NullPointerException or other runtime failures.

## Data Pipeline
The data flow for this component is initiated by user input and results in the modification of either a network channel or a global system flag.

**Latency Simulation Flow:**
> User Input (`/net latsim set...`) -> Command System Parser -> **NetworkCommand.LatencySimulationCommand.Set.execute()** -> PlayerRef lookup -> LatencySimulationHandler.setLatency() -> Player's Netty Channel Pipeline -> Delayed Packet Transmission

**Global Flag Toggle Flow:**
> User Input (`/net serverknockback`) -> Command System Parser -> **NetworkCommand.ServerKnockbackCommand.executeSync()** -> Direct mutation of `KnockbackSystems.ApplyPlayerKnockback.DO_SERVER_PREDICTION` -> Modified behavior in subsequent knockback physics calculations

