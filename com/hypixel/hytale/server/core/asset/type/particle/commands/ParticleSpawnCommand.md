---
description: Architectural reference for ParticleSpawnCommand
---

# ParticleSpawnCommand

**Package:** com.hypixel.hytale.server.core.asset.type.particle.commands
**Type:** Managed Singleton

## Definition
```java
// Signature
public class ParticleSpawnCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The **ParticleSpawnCommand** class serves as a high-level bridge between the server's command system and the world's particle rendering system. It is not responsible for the low-level logic of particle simulation or networking; instead, it orchestrates the process by parsing user intent, gathering necessary world state, and delegating the core work to specialized utilities.

This class extends **AbstractTargetPlayerCommand**, a base class that provides the foundational logic for commands that must target a specific player entity. This inheritance offloads the responsibility of parsing player selectors (e.g., @p, @a, or a player's name) and resolving them to a valid entity reference.

The command's primary function is to trigger a particle effect at a target player's location. To achieve this, it performs three critical steps within its **execute** method:
1.  **Argument Resolution:** It retrieves the specific **ParticleSystem** asset requested by the user from the **CommandContext**.
2.  **State Acquisition:** It queries the **EntityStore** for the target player's **TransformComponent** to get their precise world position and rotation.
3.  **Delegation:** It invokes **ParticleUtil.spawnParticleEffect**, passing the particle ID, transform data, and a list of nearby players who should be notified of the effect.

A key architectural pattern demonstrated here is the use of a nested subcommand, **ParticleSpawnPageCommand**. This variant allows the command to serve a dual purpose: it can either spawn a particle directly or open a server-driven UI page (**ParticleSpawnPage**) for the player, providing a graphical interface for the same action.

### Lifecycle & Ownership
-   **Creation:** A single instance of **ParticleSpawnCommand** is instantiated by the server's command registration system during the server bootstrap sequence. It is discovered, registered, and held in memory.
-   **Scope:** The object's lifecycle is tied directly to the server session. It persists from server start to server stop.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the server is shutting down and the central command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is effectively stateless with respect to command execution. Its only instance field, **particleSystemArg**, is a final definition object that describes a required command argument. This field is initialized once in the constructor and is not mutated thereafter. All data relevant to a specific execution is passed as parameters to the **execute** method.

-   **Thread Safety:** The **ParticleSpawnCommand** instance is thread-safe. As it holds no mutable state, multiple threads could theoretically invoke its logic without causing data corruption. However, it operates within the server's single-threaded or well-defined multi-threaded model. The call to **SpatialResource.getThreadLocalReferenceList** is a critical indicator of this design, as it uses a thread-local object pool to prevent heap allocations and avoid race conditions during spatial queries. All interactions with world state via the **Store** parameter are expected to occur on the main server thread.

## API Surface
The public contract is defined by the command system's invocation pattern, which calls the overridden **execute** method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | protected void | O(N) | Triggers the particle spawn logic. Complexity is O(N) where N is the number of entities within the 75-unit spatial query radius. |
| ParticleSpawnPageCommand.execute(...) | protected void | O(1) | Triggers the opening of a custom UI page for the target player. |

## Integration Patterns

### Standard Usage
This command is not intended to be used directly via code. It is invoked by the server's command handler in response to user input from the console or chat.

```
// User input that triggers this command's execution
/particle spawn PlayerName hytale:fire_large
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using **new ParticleSpawnCommand()** in game logic. The command system manages the lifecycle of all command objects. Direct instantiation will result in a non-functional object that is not registered to handle user input.
-   **Direct Invocation:** Do not call the **execute** method directly. Bypassing the command system's dispatcher will skip critical steps like permission checking, argument parsing, and context setup, leading to **NullPointerException** or other undefined behavior.

## Data Pipeline
The flow of data for this command begins with user input and ends with a network packet sent to relevant clients. The **ParticleSpawnCommand** acts as the central orchestrator in this flow.

> Flow:
> User Input (`/particle spawn ...`) -> Command Parser -> **ParticleSpawnCommand** -> EntityStore (for Transform) -> SpatialResource (for nearby players) -> ParticleUtil -> Network Packet to Clients

