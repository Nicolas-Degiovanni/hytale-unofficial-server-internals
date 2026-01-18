---
description: Architectural reference for ImageImportCommand
---

# ImageImportCommand

**Package:** com.hypixel.hytale.builtin.buildertools.imageimport
**Type:** Transient

## Definition
```java
// Signature
public class ImageImportCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The ImageImportCommand class is a server-side implementation of the Command Pattern, designed to be discovered and managed by the server's central command system. It serves as the entry point for the "Image Import" builder tool, triggered when a player executes the `/importimage` command in chat.

Architecturally, this class acts as a simple bridge. Its sole responsibility is to translate a player's command execution into a UI-centric action. It does not contain any of the core logic for image processing or world modification. Instead, it delegates this responsibility by instantiating and opening the ImageImportPage, a specialized UI component, for the executing player. This design adheres to the Single Responsibility Principle, cleanly separating command parsing from UI interaction and business logic.

This command is permission-gated, requiring Creative game mode, which is configured during its construction. It interacts with the server's Entity Component System (ECS) via the Store and Ref parameters in its execute method to retrieve the target Player component.

## Lifecycle & Ownership
- **Creation:** An instance of ImageImportCommand is created by the server's command registration service during the server bootstrap sequence or when its containing module is loaded. The system scans for classes extending AbstractPlayerCommand and instantiates them for registration.
- **Scope:** The registered instance is a stateless handler that persists for the entire server session. It is shared and reused for every execution of the `/importimage` command.
- **Destruction:** The object is marked for garbage collection when the server shuts down or its module is unloaded, at which point it is de-registered from the command system.

## Internal State & Concurrency
- **State:** This class is **stateless**. All configuration, such as the command name and permission group, is defined at construction and is immutable thereafter. All data required for execution is provided as arguments to the `execute` method.
- **Thread Safety:** The class is inherently thread-safe due to its stateless and immutable nature. However, the `execute` method operates on shared, mutable state (World, EntityStore). The server's core game loop is responsible for ensuring that command execution is synchronized correctly, typically by processing commands on the main world thread to prevent race conditions with other game logic.

## API Surface
The public contract is defined by its constructor for registration and the overridden `execute` method for invocation by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | protected void | O(1) | Retrieves the Player component for the command issuer and instructs their PageManager to open the ImageImportPage. This is the primary entry point invoked by the command dispatcher. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class directly. It is invoked automatically by the server's command handler when a player executes the corresponding chat command. The system is responsible for providing the correct context.

*Conceptual system-level registration:*
```java
// During server startup, the command system discovers and registers the command.
CommandRegistry registry = server.getCommandRegistry();
registry.register(new ImageImportCommand());

// A developer never writes the code above or below.
// The system handles invocation automatically.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ImageImportCommand()` in game logic. Creating an instance does not register it with the server; it will have no effect. Registration is a system-level concern.
- **Direct Invocation:** Never call the `execute` method directly. Doing so bypasses critical infrastructure, including permission validation, context population, and thread safety guarantees provided by the command dispatcher.

## Data Pipeline
The command acts as a trigger in a user-initiated data flow, converting a chat command into a UI event.

> Flow:
> Player Chat Input (`/importimage`) -> Server Network Layer -> Command Parser -> Command Dispatcher -> **ImageImportCommand.execute()** -> Player.PageManager.openCustomPage() -> Server UI System -> Network Packet (UI Open) -> Client UI Render

