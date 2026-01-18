---
description: Architectural reference for NPCRunTestsCommand
---

# NPCRunTestsCommand

**Package:** com.hypixel.hytale.server.npc.commands
**Type:** Transient

## Definition
```java
// Signature
public class NPCRunTestsCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts

The NPCRunTestsCommand is a server-side administrative command responsible for driving a stateful, multi-step testing sequence for Non-Player Character (NPC) roles. Architecturally, it functions as a state machine controller, where the state itself is not stored within the command object but is externalized into a dedicated Entity-Component-System (ECS) component.

The core design principle is the separation of the command's execution logic from the test session's state. The NPCRunTestsCommand class itself is stateless. When a player initiates a test sequence, a companion component, **NPCTestData**, is created and attached directly to the player's entity. This component persists across multiple invocations of the command, holding critical information such as the list of NPC roles to test, the current position in the sequence, and the results of completed tests.

This approach allows a single command, `/runtests`, to serve multiple functions based on context and arguments:
1.  **Initiation:** When run with a list of roles (or a preset), it creates the NPCTestData component and spawns the first NPC.
2.  **Progression:** When run with flags like *pass* or *fail*, it reads the existing NPCTestData component, records the result, cleans up the current NPC, and spawns the next one in the sequence.
3.  **Termination:** When run with the *abort* flag or after the final test, it generates a summary report, cleans up the last NPC, and destroys the NPCTestData component.

This command acts as a critical bridge between the server's command system, the NPC spawning logic encapsulated in NPCPlugin, and the underlying ECS data store.

### Lifecycle & Ownership
-   **Creation:** The NPCRunTestsCommand object is instantiated and registered by the server's command system during server initialization. The state-bearing **NPCTestData** component is created and attached to the player's entity by this command during the *first* invocation of a new test sequence.
-   **Scope:** The command object is a long-lived singleton managed by the command framework. The NPCTestData component is session-scoped; it lives on the player's entity for the duration of a single test run, from initiation to completion or abortion.
-   **Destruction:** The NPCTestData component is explicitly removed from the player's entity when the test sequence concludes naturally or is terminated with the *abort* flag. This cleanup is managed internally by the command's logic.

## Internal State & Concurrency
-   **State:** The NPCRunTestsCommand class is immutable after construction. Its fields are final argument definitions. All mutable state related to an active test (e.g., current NPC, failed tests) is exclusively managed within the NPCTestData component, external to this class.
-   **Thread Safety:** This class is **not thread-safe**. It is designed to be executed exclusively on the main server thread as part of the game's tick loop. All interactions with the ECS Store must be synchronized through the main loop, and this command adheres to that contract. Unmanaged, multi-threaded access will lead to data corruption and server instability.

## API Surface
The public contract is fulfilled by the `execute` method, inherited from AbstractPlayerCommand and invoked by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(N) | The primary entry point. Manages the test state machine. Complexity is O(N) during initialization, where N is the number of roles provided. Subsequent calls are O(1). |

## Integration Patterns

### Standard Usage
The command is designed for interactive use by a server administrator. The typical workflow is a sequence of commands entered into the game's chat console.

1.  **Start a test run with a preset list of NPCs:**
    ```
    /runtests preset
    ```
2.  **Observe the spawned NPC. Mark the test as passed and spawn the next one:**
    ```
    /runtests pass
    ```
3.  **Mark the current test as failed and proceed:**
    ```
    /runtests fail
    ```
4.  **End the test sequence prematurely:**
    ```
    /runtests abort
    ```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance with `new NPCRunTestsCommand()`. The command system is solely responsible for the lifecycle of command objects.
-   **Manual State Manipulation:** Do not manually add, modify, or remove the NPCTestData component from a player entity. The command's internal logic is the single source of truth for managing this component's lifecycle. Manual changes will corrupt the test state machine.
-   **Invoking `execute` Directly:** Bypassing the server's command processing system by calling the execute method from other code is unsupported. This will result in a missing or improperly configured CommandContext.

## Data Pipeline
The flow of data and control is determined by whether a test is being initiated or progressed.

**Initial Invocation Flow:**
> Player Chat Input (`/runtests roles=...`) -> Command Parser -> **NPCRunTestsCommand.execute()** -> Creates and attaches `NPCTestData` component to player entity -> `NPCPlugin.spawnEntity()` -> New NPC entity created in World -> Message sent to Player

**Progression Invocation Flow (`pass`/`fail`):**
> Player Chat Input (`/runtests pass`) -> Command Parser -> **NPCRunTestsCommand.execute()** -> Reads existing `NPCTestData` from player entity -> `World.getEntityRef()` -> Finds current NPC -> `cleanupNPC()` -> Current NPC entity removed -> `spawnNPC()` -> Next NPC entity created -> `NPCTestData` state is updated -> Message sent to Player

**Termination Flow:**
> Player Chat Input (`/runtests abort`) -> Command Parser -> **NPCRunTestsCommand.execute()** -> Reads `NPCTestData` -> `reportResults()` -> Message sent to Player -> `cleanupNPC()` -> Final NPC removed -> `Store.removeComponent()` -> `NPCTestData` is destroyed

