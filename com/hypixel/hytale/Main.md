---
description: Architectural reference for Main
---

# Main

**Package:** com.hypixel.hytale
**Type:** Utility / EntryPoint

## Definition
```java
// Signature
public final class Main {
```

## Architecture & Concepts
The Main class serves as the primary entry point for the entire Hytale application. Its role is not to initialize the game engine itself, but to perform critical, low-level bootstrapping and environment setup before any core game logic is executed.

The central architectural function of this class is to facilitate a pluggable class transformation system. It inspects the environment for "early plugins" which may need to modify game code at the bytecode level. Based on the presence of these plugins, it makes a pivotal decision:

1.  **Standard Launch:** If no class transformers are registered, it directly transfers control to the LateMain class, which proceeds with the normal application startup.
2.  **Transformed Launch:** If transformers are present, it constructs a custom TransformingClassLoader. This new class loader becomes the context class loader for the main thread. It then uses this custom loader to load and invoke the LateMain class. This ensures that any subsequent class loaded by the application can be intercepted and modified on-the-fly by the early plugins.

This two-stage launch sequence (Main -> LateMain) is a deliberate design pattern. It cleanly separates the complex and sensitive class loading modifications from the main application logic, reducing complexity and potential for errors within the core engine.

### Lifecycle & Ownership
- **Creation:** This class is never instantiated. It contains only static methods and is loaded by the Java Virtual Machine at application start to invoke its main method.
- **Scope:** The active lifecycle of the Main class is extremely brief, lasting only for the duration of the initial bootstrap sequence. Its responsibilities are concluded the moment it successfully invokes the lateMain method in the LateMain class.
- **Destruction:** The class remains loaded in the JVM for the application's lifetime but performs no further actions after the initial handoff.

## Internal State & Concurrency
- **State:** The Main class is entirely stateless. It contains no member fields and its behavior is determined solely by the command-line arguments and the state of the EarlyPluginLoader.
- **Thread Safety:** This class is inherently thread-safe due to its stateless nature. However, it is designed and expected to be executed only once, on the application's main thread, at startup. Concurrency is not a relevant concern for its intended use.

## API Surface
The public contract is limited to the standard Java entry point.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| main(String[] args) | void | O(N) | The application entry point. Initializes the early plugin system and determines the correct class loading strategy before handing off control to LateMain. Complexity is dependent on classpath scanning. |

## Integration Patterns

### Standard Usage
This class is not intended to be used by developers. It is invoked by the Java Virtual Machine when the application is launched from the command line.

```shell
# The JVM is the sole "user" of this class.
java -jar Hytale.jar
```

### Anti-Patterns (Do NOT do this)
- **Direct Invocation:** Do not call Main.main from any other part of the application code. It is designed exclusively for JVM-level bootstrapping and re-invoking it will lead to unpredictable and catastrophic state corruption.
- **Instantiation:** The class is declared as final and has no public constructor. Attempting to instantiate it via reflection or other means is unsupported and will break the application.

## Data Pipeline
The primary flow through this component is one of control, not data. It directs the application's startup path based on the presence of class transformers.

> Flow:
> JVM Start -> **Main.main(args)** -> EarlyPluginLoader -> [Decision: Transformers Present?]
>
> **Path A (No Transformers):** -> LateMain.lateMain(args) -> Game Engine Initialization
>
> **Path B (Transformers Present):** -> TransformingClassLoader Creation -> LateMain.lateMain(args) via Reflection -> Game Engine Initialization

