---
description: Architectural reference for ConsoleModule
---

# ConsoleModule

**Package:** com.hypixel.hytale.server.core.console
**Type:** Singleton Plugin

## Definition
```java
// Signature
public class ConsoleModule extends JavaPlugin {
```

## Architecture & Concepts
The ConsoleModule is a core server plugin that bridges the physical server console (standard input/output) with the server's internal command system. It is the primary mechanism for server operators to perform administrative actions via a terminal.

This module leverages the JLine library to provide a rich, interactive console experience, including support for colored text and a proper command prompt, which is a significant improvement over basic standard I/O.

Its core architectural function is to offload the blocking operation of reading console input from the main server thread. It accomplishes this by spawning a dedicated background thread, named ConsoleThread, which continuously listens for user input. When a line of text is submitted, this thread processes the input and dispatches it as a command to the global CommandManager. This design ensures that the main game loop is never stalled waiting for an administrator to type a command.

Furthermore, ConsoleModule configures the HytaleLogger backend, HytaleConsole, to use the JLine Terminal. This integration allows server log messages to be rendered correctly within the interactive console environment, respecting colors and formatting.

## Lifecycle & Ownership
- **Creation:** The ConsoleModule is instantiated by the server's plugin loader during the initial bootstrap sequence. As a core plugin, its lifecycle is directly managed by the HytaleServer instance. The static singleton instance is set within the setup method, which is invoked by the plugin framework.
- **Scope:** The module is a session-scoped singleton. It is created once when the server starts and persists until the server shuts down. The static `get()` method provides global access to this single instance throughout the server's lifetime.
- **Destruction:** The `shutdown` method is invoked by the plugin framework during the server shutdown process. This method is responsible for gracefully releasing resources, which includes closing the JLine Terminal to restore the user's original terminal settings and interrupting the dedicated ConsoleThread to terminate the input loop.

## Internal State & Concurrency
- **State:** ConsoleModule is a stateful component. It manages several key pieces of mutable state:
    - **instance:** A static reference to the singleton object.
    - **terminal:** A JLine Terminal object, which represents the connection to the physical console and its capabilities. This is a managed, stateful resource.
    - **consoleRunnable:** A reference to the Runnable that contains the logic for the dedicated console input thread.

- **Thread Safety:** This class is **not thread-safe** and is designed to be manipulated only by the main server thread during its lifecycle transitions (setup and shutdown). The primary concurrency pattern is the delegation of blocking I/O to the internal ConsoleRunnable, which runs on its own ConsoleThread.

    Communication from the ConsoleThread back to the server's main logic is achieved by invoking `CommandManager.get().handleCommand`. The CommandManager is expected to be thread-safe or to delegate command execution to the main server thread to prevent race conditions within game state. Direct access to the ConsoleModule's internal fields from arbitrary threads is unsafe.

## API Surface
The public API is minimal, primarily exposing the singleton instance and the underlying terminal for potential integration with other systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | ConsoleModule | O(1) | Statically retrieves the singleton instance of the module. |
| getTerminal() | Terminal | O(1) | Returns the underlying JLine Terminal instance. |

## Integration Patterns

### Standard Usage
Direct interaction with the ConsoleModule from other plugins is uncommon, as its primary function is to feed the global CommandManager. However, another system could retrieve the instance to gain access to the JLine terminal for advanced console output.

```java
// Retrieve the singleton instance
ConsoleModule console = ConsoleModule.get();

// Access the underlying terminal (for advanced use cases)
Terminal terminal = console.getTerminal();
terminal.writer().println("Custom message written directly to the console.");
terminal.writer().flush();
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ConsoleModule()`. The server's plugin loader is solely responsible for its creation and lifecycle management. Attempting to create it manually will result in a broken, non-functional instance and may destabilize the server.
- **Manual Lifecycle Management:** Do not call the `setup` or `shutdown` methods directly. These are framework-level callbacks and invoking them manually will lead to resource leaks or crashes.
- **Unsynchronized Terminal Access:** The JLine Terminal object is not inherently thread-safe. Accessing it from multiple threads without proper synchronization can corrupt the console's state, leading to garbled output.

## Data Pipeline
The ConsoleModule implements a simple but critical data pipeline for processing administrative commands entered into the server console.

> Flow:
> User Input (stdin) -> JLine LineReader -> **ConsoleRunnable** (String Sanitization) -> CommandManager -> Command Execution

