---
description: Architectural reference for NPCSensorStatsCommand
---

# NPCSensorStatsCommand

**Package:** com.hypixel.hytale.server.npc.commands
**Type:** Transient

## Definition
```java
// Signature
public class NPCSensorStatsCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The NPCSensorStatsCommand is a server-side diagnostic tool designed for developers and content creators. It is not part of the core gameplay loop but serves as a critical utility for auditing the sensory configurations of all defined Non-Player Character (NPC) roles.

Its primary function is to perform a comprehensive "dry run" of the instantiation process for every available NPC role template. For each role, it simulates the build process to extract and aggregate metadata about its sensory systems, such as entity detection ranges, player avoidance radii, and other behavioral triggers.

This command operates by interfacing directly with the NPCPlugin, which acts as the central service for the entire NPC system. The command's output is intentionally directed to the server logs rather than the in-game chat console, reinforcing its status as a backend debugging and validation utility. The core mechanism involves populating a transient RoleStats object during a simulated role build, which acts as a temporary data container for the sensor metrics.

**WARNING:** This command is computationally expensive. It iterates and simulates the creation of every single NPC role defined on the server. Execution on a live server with a large number of roles will cause a significant, albeit temporary, performance degradation and should be performed only during maintenance or low-traffic periods.

### Lifecycle & Ownership
-   **Creation:** A single instance of NPCSensorStatsCommand is instantiated by the server's command registration system during server bootstrap. It is registered under the name *sensorstats*.
-   **Scope:** The command object persists for the entire lifetime of the server process. However, the execution context and all objects generated within the *execute* method are ephemeral.
-   **Destruction:** The singleton instance is destroyed when the server shuts down. All temporary objects created during execution, most notably the temporary NPCEntity and the per-role Role instances, are garbage collected at the end of the *execute* method's scope. The command explicitly calls *remove()* on the temporary NPC to ensure immediate and clean despawning.

## Internal State & Concurrency
-   **State:** The NPCSensorStatsCommand class is entirely stateless. All operational data, such as the list of roles and the formatted output string, is stored in local variables within the *execute* method. This design ensures that each invocation is completely isolated from any previous runs.

-   **Thread Safety:** This command is **not thread-safe**. It is designed to be executed exclusively on the main server thread by the CommandSystem. Its operations, which involve entity spawning and component access, are fundamentally tied to the server's single-threaded game loop. Concurrent execution would lead to race conditions and world state corruption.

## API Surface
The programmatic surface is limited to the contract defined by its parent, AbstractPlayerCommand. The primary interaction is through the server's command dispatcher.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(N * M) | Triggers the analysis of all NPC roles. Complexity is high, where N is the total number of role templates and M is the average complexity of building a single role. This is a blocking operation that can stall the server thread. |

## Integration Patterns

### Standard Usage
This command is intended to be executed by a server administrator or developer via the in-game console to generate a report on NPC sensory data.

```java
// This command is not intended for programmatic invocation.
// It is run by a player or through the server console.
// Example: /sensorstats
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using *new NPCSensorStatsCommand()*. The command system manages the lifecycle of this object.
-   **Programmatic Invocation:** Do not call the *execute* method directly from other parts of the server codebase. This bypasses the command system's context and permission handling.
-   **Production Usage:** Avoid running this command on a high-population production server. The performance impact of iterating through hundreds of roles can cause severe server lag or unresponsiveness.

## Data Pipeline
The command initiates a data-gathering and reporting pipeline that transforms NPC role definitions into a formatted log output. It does not process continuous data streams but rather performs a one-shot batch analysis.

> Flow:
> Command Input -> CommandSystem -> **NPCSensorStatsCommand** -> NPCPlugin (Role Builder Simulation) -> RoleStats (In-Memory Aggregator) -> Formatted StringBuilder -> Server Logger

