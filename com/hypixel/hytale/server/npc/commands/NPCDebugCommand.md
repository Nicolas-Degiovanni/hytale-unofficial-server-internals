---
description: Architectural reference for NPCDebugCommand
---

# NPCDebugCommand

**Package:** com.hypixel.hytale.server.npc.commands
**Type:** Utility

## Definition
```java
// Signature
public class NPCDebugCommand extends AbstractCommandCollection {
```

## Architecture & Concepts

The NPCDebugCommand class serves as a command aggregator within the server's core command processing system. It provides a centralized developer and administrator interface for manipulating the real-time behavior of Non-Player Characters (NPCs) by modifying their associated `RoleDebugFlags`. This class does not implement any debugging logic itself; rather, it acts as a container and entry point for a suite of specialized sub-commands.

Architecturally, this class follows the Composite pattern. It is a node in the server's command tree, registered under the primary `npc` command, and it contains leaf nodes representing the actual operations: `set`, `show`, `toggle`, `clear`, `defaults`, and `presets`. Each sub-command is implemented as a nested static class, promoting high cohesion and clear separation of concerns.

A critical aspect of its design is the direct integration with the server's Entity Component System (ECS). When a debug flag is modified, the system invokes `safeSetRoleDebugFlags`, which not only updates the state on the `NPCEntity` but also explicitly removes the `Nameplate` component from the entity's `EntityStore`. This action forces the engine to re-evaluate and re-render the NPC's nameplate, ensuring that any visual indicators tied to debug flags are updated immediately.

## Lifecycle & Ownership
- **Creation:** A single instance of NPCDebugCommand is created by the server's command registration system during the server bootstrap sequence. It is then registered as a sub-command of a higher-level `NPCCommand` collection.
- **Scope:** Application-scoped. The instance persists for the entire lifetime of the server process. It is designed to be a long-lived, stateless object.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection only upon server shutdown, when the central command registry is cleared.

## Internal State & Concurrency
- **State:** The NPCDebugCommand class and its nested sub-commands are fundamentally stateless. They do not maintain any session-specific or mutable data. All state modifications are performed on external `NPCEntity` objects passed into the `execute` methods via the `CommandContext`.
- **Thread Safety:** This class is not thread-safe and is not designed to be. Command execution is orchestrated by the server's main thread or a dedicated command processing thread. All interactions with game state, such as modifying an `NPCEntity` or its components in the `EntityStore`, must be synchronized with the main server game loop.

**WARNING:** Never invoke command execution methods from an asynchronous thread without first queuing the operation to be executed on the main server thread. Direct modification of game state from other threads will lead to race conditions, data corruption, and server instability.

## API Surface

The primary API of this class is not programmatic but rather the command-line interface it exposes to server operators. The following sub-commands constitute its public contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| show | Sub-Command | O(N) | Displays the current set of `RoleDebugFlags` for the selected NPC(s). |
| set | Sub-Command | O(N) | Overwrites the existing `RoleDebugFlags` with a new, specified set of flags. |
| toggle | Sub-Command | O(N) | Adds or removes the specified `RoleDebugFlags` from the existing set on the NPC(s). |
| clear | Sub-Command | O(N) | Removes all `RoleDebugFlags` from the selected NPC(s). |
| defaults | Sub-Command | O(N) | Resets the `RoleDebugFlags` on the selected NPC(s) to a predefined default preset. |
| presets | Sub-Command | O(1) | Lists all available debug flag presets or shows the flags within a specific preset. |

*Complexity is O(N) where N is the number of entities targeted by the command selector.*

## Integration Patterns

### Standard Usage

This class is intended to be used via the in-game console or chat by an administrator. It is not meant to be invoked programmatically by other game systems.

```sh
# Example: Set specific debug flags on a targeted NPC
/npc debug set @e[type=npc,limit=1] pathing,combat_logic

# Example: Toggle the visibility of an NPC's navigation path
/npc debug toggle @e[type=npc,limit=1] pathing

# Example: Clear all debug flags
/npc debug clear @e[type=npc,limit=1]
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new NPCDebugCommand()`. The command system is responsible for the lifecycle of all command objects. Manually creating an instance will result in a non-functional command that is not registered with the server.
- **Programmatic Execution:** Avoid calling the `execute` methods on the sub-commands directly. The `CommandContext` is a complex object built by the command dispatcher. Bypassing the dispatcher will result in a missing or incomplete context, leading to `NullPointerException` and unpredictable behavior.

## Data Pipeline

The flow of data for a typical debug operation is initiated by a user and terminates with a change in NPC state and a visual update in the game world.

> Flow:
> User Input (`/npc debug set...`) -> Server Network Layer -> Command Parser -> **NPCDebugCommand** -> SetCommand.execute() -> `safeSetRoleDebugFlags` -> NPCEntity state is mutated -> `EntityStore` removes Nameplate component -> NPC Behavior System reads new flags -> Game Renderer updates visuals

---

