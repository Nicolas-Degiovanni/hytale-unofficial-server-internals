---
description: Architectural reference for EmoteCommand
---

# EmoteCommand

**Package:** com.hypixel.hytale.server.core.cosmetics.commands
**Type:** Transient Handler

## Definition
```java
// Signature
public class EmoteCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The EmoteCommand class is a concrete implementation of the Command Pattern, designed to handle player-initiated emote actions on the server. It serves as a critical link between the server's command processing system and the underlying cosmetics and animation game logic.

Its primary function is to parse a player's text command, validate the requested emote against the central CosmeticRegistry, and trigger the corresponding animation on the player's entity. This class effectively decouples the raw user input from the game mechanics, allowing the command system to remain agnostic about how an emote is performed. The system identifies the command string "emote", routes the request to this handler, and relies on its `execute` method to interface with the appropriate game systems.

## Lifecycle & Ownership
- **Creation:** A single instance of EmoteCommand is instantiated by the server's command registration system during the server bootstrap phase. The system scans for command handler classes and registers them in a central lookup table.
- **Scope:** The EmoteCommand object is a long-lived, stateless handler. It persists for the entire server session. The state relevant to a specific execution, such as the player and the arguments, is passed in via the CommandContext and is scoped only to the invocation of the `execute` method.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the server shuts down or the command module is unloaded.

## Internal State & Concurrency
- **State:** This class is fundamentally stateless. Its only member field, `emoteArg`, is an immutable definition for a required command argument. It is configured once during construction and never changes. All transactional state is provided externally to the `execute` method.
- **Thread Safety:** The `execute` method is designed to be invoked by the server's main thread or a world-specific update thread. It is not thread-safe for concurrent invocations on the same player entity. The operations it performs on the World and EntityStore are assumed to be part of a single-threaded game tick, making external synchronization unnecessary under normal operating conditions.

## API Surface
The public contract is defined by its parent, AbstractPlayerCommand. The core logic is contained entirely within the `execute` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Executes the emote logic. Looks up the emote in a map. On failure, may perform a more complex O(N) fuzzy search for suggestions. |

## Integration Patterns

### System Invocation
This class is not intended for direct use by developers. It is invoked exclusively by the server's command system in response to a player's chat input. The system handles parsing, permission checks, and context creation before delegating to this handler.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new EmoteCommand()`. The command system manages the lifecycle of command handlers. Direct instantiation will result in an object that is not registered to receive commands.
- **Manual Invocation:** Do not call the `execute` method directly. Bypassing the command system's invocation pipeline will skip critical steps like permission validation and argument parsing, potentially leading to server instability or security vulnerabilities.

## Data Pipeline
The flow of data for an emote command is linear, originating from the client and resulting in an animation state change that is broadcast back out.

> Flow:
> Player Chat Input (`/emote wave`) -> Network Packet -> Server Command Parser -> **EmoteCommand** -> CosmeticRegistry -> AnimationUtils -> EntityStore Update -> World State Change -> Network Broadcast to Clients

