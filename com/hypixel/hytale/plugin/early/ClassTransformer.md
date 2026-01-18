---
description: Architectural reference for ClassTransformer
---

# ClassTransformer

**Package:** com.hypixel.hytale.plugin.early
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface ClassTransformer {
   // ... methods
}
```

## Architecture & Concepts
The ClassTransformer interface defines a low-level contract for the Hytale plugin system, enabling deep modification of the game's core behavior. It is a key component of the "early" plugin loading phase, which occurs during the Java Virtual Machine's class loading process, well before the main game engine is initialized.

Implementations of this interface, known as *transformers*, are granted the ability to intercept and rewrite the bytecode of any class as it is being loaded by the game. This mechanism is the foundation for advanced modding capabilities, allowing plugins to alter game logic, add new features, or fix bugs directly within the core engine classes.

The system operates as a chain of responsibility. When the Hytale ClassLoader loads a class, it passes the raw bytecode through a sequence of all registered ClassTransformer instances. Each transformer can inspect, modify, or completely replace the bytecode before passing it to the next transformer in the chain. The `priority` method is used to establish a deterministic order for this chain, preventing conflicts between plugins that wish to modify the same class.

**WARNING:** This is a highly privileged and sensitive API. Errors within a transformer, such as producing invalid bytecode or performing slow operations, can prevent the game from starting or introduce severe performance issues and instability.

## Lifecycle & Ownership
- **Creation:** Implementations are not instantiated directly by developers. Instead, the Hytale plugin loader discovers them on the classpath at startup, typically using a service discovery mechanism (e.g., Java ServiceLoader). The loader is responsible for creating a single instance of each discovered transformer.
- **Scope:** A ClassTransformer instance exists for the duration of the application's class loading phase. It is invoked repeatedly for different classes as they are loaded by the JVM. The instance itself is typically held in a central registry managed by the plugin system for the entire application lifetime.
- **Destruction:** Instances are managed by the plugin loader. They are eligible for garbage collection when the plugin system is shut down or the owning plugin is unloaded, which generally coincides with the termination of the game client or server.

## Internal State & Concurrency
- **State:** Implementations of ClassTransformer **must be stateless**. The `transform` method should operate as a pure function, deriving its output solely from its inputs. Storing mutable state within a transformer instance is a critical anti-pattern that can lead to memory leaks, unpredictable behavior, and severe concurrency issues.
- **Thread Safety:** The JVM may load classes on multiple threads concurrently to improve startup performance. Therefore, all implementations of the `transform` method **must be thread-safe**. Any shared resources accessed must be protected, though the recommended and safest approach is to avoid shared state entirely.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| priority() | int | O(1) | Returns the execution priority. Transformers are sorted in descending order of priority. Higher values execute earlier in the transformation chain. |
| transform(name, transformedName, bytes) | byte[] | O(N) | The core transformation callback. Receives the class name and its bytecode. Returns the modified bytecode. Returning null or the original byte array signals that no transformation was performed. |

## Integration Patterns

### Standard Usage
A plugin developer does not call methods on this interface. Instead, they provide an implementation that is automatically discovered and invoked by the engine.

1.  Create a class that implements the ClassTransformer interface.
2.  Implement the `transform` method to contain bytecode manipulation logic (e.g., using a library like ASM or Javassist).
3.  Implement the `priority` method to declare its position in the execution chain.
4.  Register the implementation as a service, typically via a `META-INF/services` file, so the Hytale plugin loader can discover it.

```java
// Example of a simple transformer implementation
// This code is NOT called directly. It is invoked by the engine.
public class MyExampleTransformer implements ClassTransformer {

    @Override
    public int priority() {
        return 100; // A custom priority
    }

    @Nullable
    @Override
    public byte[] transform(@Nonnull String className, @Nonnull String transformedName, @Nonnull byte[] classBytes) {
        // Only transform a specific target class
        if (!"com.hypixel.hytale.game.SomeGameSystem".equals(className)) {
            return null; // Return null to skip transformation
        }

        // ... logic to modify the classBytes using a bytecode library ...
        byte[] modifiedBytes = modifyBytecode(classBytes);

        return modifiedBytes;
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Storing data in instance fields across multiple `transform` calls is extremely dangerous and will lead to race conditions and non-deterministic behavior.
- **Blocking Operations:** Never perform file I/O, network requests, or any other long-running, blocking operations within the `transform` method. Doing so will dramatically increase game startup time and may cause deadlocks in the class loading system.
- **Ignoring Priority:** Failing to provide a meaningful priority can lead to unpredictable interactions with other plugins that may be transforming the same classes.

## Data Pipeline
The flow of data through the transformation system is linear and synchronous within the context of a single class being loaded.

> Flow:
> JVM Class Load Request -> Hytale ClassLoader -> Plugin Service Loader -> **All ClassTransformer Instances (Sorted by Priority)** -> Final Bytecode -> JVM Class Definition

