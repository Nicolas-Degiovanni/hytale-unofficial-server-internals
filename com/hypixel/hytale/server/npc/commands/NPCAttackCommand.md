---
description: Architectural reference for NPCAttackCommand
---

# NPCAttackCommand

**Package:** com.hypixel.hytale.server.npc.commands
**Type:** Transient Configuration

## Definition
```java
// Signature
public class NPCAttackCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The NPCAttackCommand class is a command collection, serving as a container for related server-side console commands that manipulate the combat behavior of Non-Player Characters (NPCs). It adheres to the Command design pattern, providing a high-level, user-facing interface for administrators and scripters to dynamically alter an NPCEntity's attack patterns.

This class itself contains no operational logic. Its sole responsibility is to register a primary command, *attack*, and delegate execution to its nested sub-commands:
*   **SetAttackCommand:** Overwrites an NPC's default attack AI with a specific, ordered list of Interaction assets.
*   **ClearAttackCommand:** Removes any active attack overrides, restoring the NPC's default, AI-driven combat behavior.

The core mechanism of this system is the modification of the **CombatSupport** component, which is part of an NPCEntity's role definition. By populating or clearing a list of *attack overrides* within this component, the command effectively hijacks the NPC's decision-making process for combat actions. This is a critical feature for designing scripted encounters, boss fight phases, and for debugging combat mechanics without altering the NPC's base configuration files.

### Lifecycle & Ownership
-   **Creation:** An instance of NPCAttackCommand is created by the server's central command registration system during server bootstrap or when a game module containing it is loaded.
-   **Scope:** The object instance is retained by the command registry for the entire server session. While the object itself is stateless, its registration is global and persistent.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server shuts down or the parent module is unloaded.

## Internal State & Concurrency
-   **State:** The NPCAttackCommand and its inner sub-commands are **stateless**. They hold immutable definitions for command arguments but do not store any data related to a specific execution. All state modification is performed on the target NPCEntity and its components passed into the execute method.

-   **Thread Safety:** This class is **not thread-safe** for execution. The server's CommandSystem guarantees that all command execution occurs on the main server thread, which is synchronized with the world tick. Invoking the execute method from any other thread will lead to race conditions and world corruption. The object's configuration is immutable after construction, making it safe to read from multiple threads, though this has no practical application.

## API Surface
The primary interface for this class is not programmatic but through the server console. The methods below represent the logical operations performed by the sub-commands upon execution.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SetAttackCommand.execute(...) | void | O(N) | Clears existing overrides and adds a new sequence of N attacks to the target NPC's CombatSupport component. |
| ClearAttackCommand.execute(...) | void | O(1) | Removes all attack overrides from the target NPC's CombatSupport component, restoring default AI. |

## Integration Patterns

### Standard Usage
This class is designed to be invoked via the server console or through automated command blocks. It is not intended for direct programmatic use. The command targets the currently selected NPC or an NPC specified by standard entity selectors.

```
# Forces the selected NPC to use a specific two-hit combo
/npc selected attack set my_assets:sword_slash my_assets:shield_bash

# Restores the NPC's normal combat AI
/npc selected attack clear
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new NPCAttackCommand()` in game logic. The command system handles instantiation and registration. Creating an instance has no effect.
-   **Programmatic Invocation:** Do not call the `execute` methods directly. This bypasses critical infrastructure such as permission checks, argument parsing, and thread safety guarantees. To run a command programmatically, use the server's central command dispatch service.

## Data Pipeline
The flow of control for this command begins with user input and terminates with a state change in a world entity component.

> Flow:
> Console Input -> Command Parser -> CommandSystem Dispatcher -> **NPCAttackCommand** (selects sub-command) -> SetAttackCommand::execute -> NPCEntity -> CombatSupport Component -> AI Behavior System Update

