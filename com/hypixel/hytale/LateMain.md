---
description: Architectural reference for LateMain
---

# LateMain

**Package:** com.hypixel.hytale
**Type:** Utility

## Definition
```java
// Signature
public class LateMain {
```

## Architecture & Concepts
The LateMain class serves as the secondary, platform-agnostic entry point for the Hytale server application. Its sole responsibility is to orchestrate the critical server bootstrap sequence. It acts as the bridge between the operating system process invocation and the active HytaleServer instance.

This class is invoked after initial, potentially native, setup has occurred. Its primary functions are:
1.  **Command-Line Argument Parsing:** It delegates to the Options utility to interpret arguments passed at startup.
2.  **Logging Subsystem Initialization:** It configures and activates the HytaleLogger, including file handlers and a sophisticated, multi-source log level resolution strategy.
3.  **Server Instantiation:** It creates the root HytaleServer object, which subsequently takes control of the application lifecycle.
4.  **Top-Level Exception Handling:** It provides a final-resort exception handler for the entire startup process, ensuring that catastrophic failures are logged, reported to Sentry, and result in a clean process termination.

## Lifecycle & Ownership
-   **Creation:** This class is never instantiated. It contains only a static method, which is loaded by the JVM class loader upon first access. The lateMain method is invoked by the application's primary entry point.
-   **Scope:** The execution scope of the lateMain method is transient and confined to the application's startup phase. Once it successfully instantiates HytaleServer, its role is complete.
-   **Destruction:** The method's stack frame is destroyed upon completion. The class itself is unloaded when the JVM shuts down.

## Internal State & Concurrency
-   **State:** LateMain is entirely stateless. It contains no member fields and all operations are performed on method-local variables or by manipulating the state of other static subsystems like Options and HytaleLogger.
-   **Thread Safety:** This class is **not thread-safe** and is fundamentally designed for single-threaded execution. The lateMain method must only be invoked once on the main application thread at startup. Concurrent or repeated invocations will lead to unpredictable behavior and likely state corruption of the logging and server subsystems.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| lateMain(String[] args) | static void | Bootstrap | Orchestrates the entire server initialization sequence. This method will block until the server is either instantiated or a fatal error occurs. Throws RuntimeException on failure. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by module developers. It is part of the core application launcher. The standard pattern is for the JVM to invoke this method as the main entry point for the server.

```java
// Invocation by the application launcher
public class Application {
    public static void main(String[] args) {
        // ... potential native or pre-setup ...
        LateMain.lateMain(args);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Multiple Invocations:** Never call lateMain more than once during the application's lifetime. This will attempt to re-initialize static systems and create a second server instance, leading to catastrophic failure.
-   **Calling After Startup:** Do not invoke lateMain after the HytaleServer has already been initialized. The bootstrap logic is not idempotent and assumes it is running in a clean environment.
-   **Concurrent Execution:** Do not call lateMain from any thread other than the main application thread. The subsystems it configures (especially logging) are not designed for concurrent initialization.

## Data Pipeline
The data flow through LateMain is a linear sequence of configuration and instantiation events driven by command-line arguments.

> Flow:
> Command Line Arguments -> Options.parse -> **LateMain** -> HytaleLogger.init -> HytaleServer (new) -> Server Game Loop

