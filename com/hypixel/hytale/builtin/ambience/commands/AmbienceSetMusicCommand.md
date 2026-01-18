---
description: Architectural reference for AmbienceSetMusicCommand
---

# AmbienceSetMusicCommand

**Package:** com.hypixel.hytale.builtin.ambience.commands
**Type:** Transient Command

## Definition
```java
// Signature
public class AmbienceSetMusicCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The AmbienceSetMusicCommand class provides a server-side administrative command for forcefully overriding the current ambient music track within a specific world. It serves as a direct, high-level interface between a privileged user (like a server administrator) and the world's ambient audio state.

Architecturally, this class is a leaf node in the command system. It is designed to be discovered and registered at server startup. Upon invocation, it retrieves the world-specific AmbienceResource from the world's central EntityStore and mutates its state. This design decouples the command input and parsing logic from the core state management of the ambience system, allowing for a clean separation of concerns.

Its inheritance from AbstractWorldCommand ensures that it operates within the context of a valid, loaded world, preventing state corruption or attempts to modify a non-existent environment.

### Lifecycle & Ownership
- **Creation:** A single instance of AmbienceSetMusicCommand is instantiated by the server's command registration system during the bootstrap sequence. It is not created on-demand.
- **Scope:** The object instance is application-scoped and persists for the entire lifetime of the server. The execution context provided to its methods, however, is request-scoped and valid only for the duration of a single command invocation.
- **Destruction:** The instance is eligible for garbage collection only upon server shutdown when the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless. Its only instance field, ambienceFxIdArg, is a final definition object for command argument parsing, initialized at construction and never modified. All operations within the execute method are performed on external state objects passed in as parameters, primarily the world's AmbienceResource.
- **Thread Safety:** This class is **not** thread-safe for arbitrary concurrent access. The server's command execution framework is responsible for invoking the execute method from a safe thread, typically the main server thread for the target world. Calling execute from an unmanaged thread will bypass engine-level concurrency controls and will almost certainly lead to world state corruption.

## API Surface
The primary public contract is not a Java method to be called by other systems, but rather the command string and arguments it registers with the server.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setmusic | Command | O(1) | Registers the "setmusic" command. Requires one argument: the asset ID of an AmbienceFX resource. |
| execute(context, world, store) | protected void | O(1) | Internal execution logic. Retrieves the AmbienceResource and sets the forced music ID. Throws exceptions on invalid arguments. |

## Integration Patterns

### Standard Usage
This command is intended to be executed by a user through the server console or in-game chat with appropriate permissions.

```plaintext
// Sets the world's music to the specified AmbienceFX asset.
/setmusic hytale:example_dungeon_music
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new AmbienceSetMusicCommand()`. The command system manages the lifecycle of registered commands. Manual instantiation creates an unmanaged object that the server is unaware of.
- **Manual Execution:** Do not acquire the registered instance of this command and call its execute method directly. This bypasses critical infrastructure for permission checking, argument parsing, and context provision, which will result in unstable server behavior or crashes.

## Data Pipeline
The flow of data for this command is unidirectional, originating from user input and resulting in a world state change.

> Flow:
> User Console Input (`/setmusic ...`) -> Server Command Parser -> **AmbienceSetMusicCommand.execute()** -> World.EntityStore lookup -> AmbienceResource.setForcedMusicAmbience() -> World State Mutation -> (Implied) State replication to clients

