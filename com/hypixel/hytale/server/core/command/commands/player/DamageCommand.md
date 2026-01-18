---
description: Architectural reference for DamageCommand
---

# DamageCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player
**Type:** Transient

## Definition
```java
// Signature
public class DamageCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The DamageCommand class is an implementation of the **Command Object Pattern**, serving as the server-side logic for the in-game `/damage` command. Its primary architectural role is to act as a translator between raw user input and the core entity damage mechanics.

This class does not contain the logic for applying damage. Instead, it is a lightweight dispatcher responsible for:
1.  **Parsing & Validation:** Defining and parsing command arguments such as the target player, damage amount, and flags (e.g., silent).
2.  **Permission Enforcement:** Verifying that the command sender has the required permissions (`damage.self` or `damage.other`) before proceeding.
3.  **Contextual Execution:** Distinguishing between damaging the command sender and damaging another player. This is handled cleanly through the use of a nested `DamageOtherCommand` class, which encapsulates the logic for the "other player" variant.
4.  **Delegation:** Constructing a `Damage` data object and delegating the actual application of damage to the centralized `DamageSystems` module. This separation of concerns is critical, ensuring that all damage, regardless of source, is processed through a single, consistent system.

The command's structure, with a main class for the base case and a nested class for a usage variant, is a common and preferred pattern within the server's command system.

## Lifecycle & Ownership
-   **Creation:** A single instance of DamageCommand is created by the command registration system during server bootstrap. It is not instantiated per-execution.
-   **Scope:** The singleton instance persists for the entire server session. It is registered in a central command map and reused for every invocation of the `/damage` command.
-   **Destruction:** The instance is de-registered and becomes eligible for garbage collection only during server shutdown.

## Internal State & Concurrency
-   **State:** The DamageCommand object is effectively **immutable and stateless** after its initial construction. Its fields, such as `amountArg` and `silentArg`, are definitions for the argument parser and do not change during the server's lifetime. All state relevant to a specific execution is passed via the `CommandContext` parameter.

-   **Thread Safety:** The class instance is inherently thread-safe due to its stateless nature. However, its `execute` methods are designed to be invoked by the command system on a specific thread, typically the main server thread or a world-specific thread.

    **WARNING:** The nested `DamageOtherCommand` demonstrates a critical concurrency pattern. When targeting another player, the damage logic must be executed on the thread corresponding to the *target's* world. This is achieved by wrapping the core logic in a `world.execute()` call, which safely schedules the task on the correct thread, preventing race conditions and ensuring data consistency within the Entity Component System.

## API Surface
The primary public contract for this class is not a programmatic API but its registration as a chat command. The following describes the behavior initiated by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | protected void | O(1) | Entry point for the `/damage` self-variant. Parses arguments, creates a Damage object, and delegates to DamageSystems. |
| DamageOtherCommand.executeSync(...) | protected void | O(1) | Entry point for the `/damage <player>` variant. Dispatches damage logic to the target player's world thread for safe execution. |

## Integration Patterns

### Standard Usage
Developers do not instantiate or invoke this class directly. The system is designed for interaction via the in-game command parser. The following example illustrates how the system internally dispatches a parsed command to the `DamageOtherCommand` variant.

```java
// Conceptual example of the command system's invocation
CommandContext context = buildContextForPlayerInput("/damage some_player 10.0 --silent");
PlayerRef targetPlayerRef = context.getArgument("player"); // Resolves to some_player
World targetWorld = targetPlayerRef.getReference().getStore().getExternalData().getWorld();

// The system dispatches the logic to the target's world thread
targetWorld.execute(() -> {
    // ... inside DamageOtherCommand.executeSync ...
    Damage damage = new Damage(source, DamageCause.COMMAND, 10.0f);
    DamageSystems.executeDamage(targetPlayerRef.getReference(), targetWorld.getEntityStore(), damage);
});
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new DamageCommand()`. The command system manages the lifecycle of command objects. Direct instantiation serves no purpose as the object would not be registered to handle any input.
-   **Manual Invocation:** Do not call the `execute` or `executeSync` methods directly. These methods rely on a fully populated `CommandContext` and a specific thread context, which are only guaranteed when invoked by the server's command processing pipeline. Bypassing the system can lead to null pointer exceptions, incorrect thread access, and permission bypasses.

## Data Pipeline
The flow of data for this command begins with user input and terminates with an entity state change, which is then replicated to clients.

> Flow:
> Player Chat Input (`/damage ...`) → Network Packet → Server Command Parser → **DamageCommand** → Damage Object Instantiation → `DamageSystems.executeDamage` → Entity Health Component Update → State Replication to Clients

