---
description: Architectural reference for CommandUtil
---

# CommandUtil

**Package:** com.hypixel.hytale.server.core.command.system
**Type:** Utility

## Definition
```java
// Signature
public class CommandUtil {
```

## Architecture & Concepts
CommandUtil is a stateless, static utility class that provides foundational helper methods and shared constants for the server's command processing framework. It is not a service or a manager, but rather a collection of pure functions designed to enforce consistency and reduce code duplication across all command implementations.

Its primary architectural role is to centralize two key concerns:
1.  **Permission Enforcement:** Provides a single, authoritative method, requirePermission, to act as a guard clause at the entry point of command handlers. This ensures permission checks are uniform and non-bypassable.
2.  **String Manipulation:** Offers common parsing functions, like stripCommandName, to decouple raw command string processing from the business logic of the command itself.

By providing these shared tools, CommandUtil allows individual command handlers to focus solely on their specific logic, assuming that input has been sanitized and permissions have been verified in a standard way.

### Lifecycle & Ownership
-   **Creation:** As a static utility class, CommandUtil is never instantiated. Its bytecode is loaded into the JVM by the ClassLoader when it is first referenced by another class, typically a command handler or the core command dispatcher.
-   **Scope:** Application-wide. The class and its static members are available for the entire lifetime of the server process.
-   **Destruction:** The class is unloaded from the JVM when the server application shuts down. No manual cleanup is required.

## Internal State & Concurrency
-   **State:** CommandUtil is completely stateless. It contains only static final constants and pure functions whose output depends exclusively on their input arguments. It holds no mutable fields and performs no caching.
-   **Thread Safety:** This class is inherently thread-safe. Due to its stateless and immutable nature, its methods can be safely invoked concurrently from any number of threads without locks or other synchronization primitives. This is critical for a server environment where commands may be processed on multiple worker threads.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| stripCommandName(String) | String | O(N) | Parses a raw command string, returning only the arguments that follow the initial command name. N is the string length. |
| requirePermission(PermissionHolder, String) | void | O(1) | Asserts that the holder possesses the specified permission. Throws a NoPermissionException on failure. |

## Integration Patterns

### Standard Usage
CommandUtil methods should be invoked at the beginning of a command handler's execution flow to validate prerequisites before proceeding with business logic.

```java
// Inside a hypothetical command handler
public void execute(CommandContext context) {
    // Use as a guard clause to halt execution if permissions are missing.
    CommandUtil.requirePermission(context.getSender(), "hytale.command.gamemode");

    // Use to parse arguments from the raw input.
    String args = CommandUtil.stripCommandName(context.getRawCommand());
    
    // ... proceed with command logic
}
```

### Anti-Patterns (Do NOT do this)
-   **Manual Permission Checks:** Do not write your own permission checking logic. The requirePermission method is the single source of truth and is guaranteed to throw the correct exception type, which the global command exception handler is designed to catch.
    ```java
    // BAD: Redundant and inconsistent
    if (!sender.hasPermission("...")) {
        throw new RuntimeException("No permission!");
    }
    
    // GOOD: Uses the standard utility
    CommandUtil.requirePermission(sender, "...");
    ```
-   **Instantiation:** The class has no public constructor and is not designed to be instantiated. All members are static. Attempting to instantiate it via reflection will break the stateless contract.

## Data Pipeline
CommandUtil does not participate in a data pipeline as a distinct stage. Instead, its methods are invoked by other components *within* the pipeline to perform validation and transformation.

> Flow:
> Client Input -> Network Layer -> Command Dispatcher -> **CommandUtil.requirePermission()** -> Command Handler Logic

