---
description: Architectural reference for BooleanConsumer
---

# BooleanConsumer

**Package:** com.hypixel.hytale.function.consumer
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface BooleanConsumer {
   void accept(boolean var1);
}
```

## Architecture & Concepts
BooleanConsumer is a specialized functional interface designed for high-performance operations on primitive boolean values. It serves as a direct replacement for the standard Java `java.util.function.Consumer<Boolean>`, eliminating the performance overhead associated with autoboxing a primitive `boolean` into a `Boolean` wrapper object.

This interface represents a behavioral contract for any component that needs to accept a boolean value to perform a side effect, such as updating state, logging, or triggering an event. It is a foundational element of the engine's event and callback system, promoting a functional programming style in performance-critical code paths. By providing a standardized "sink" for boolean values, it decouples event producers from event consumers.

## Lifecycle & Ownership
As a functional interface, BooleanConsumer itself does not have a lifecycle. Instead, the lifecycle pertains to its *implementations*, which are typically provided as lambda expressions or method references.

- **Creation:** An implementation is created at its declaration site. This can be a lambda expression assigned to a variable, passed as a method argument, or a reference to a compatible method. The Java Virtual Machine handles the instantiation.
- **Scope:** The scope and lifetime of a BooleanConsumer implementation are tied to the object that holds a reference to it. If it is a lambda that captures variables from its enclosing scope (a closure), it will retain a reference to that scope, preventing it from being garbage collected.
- **Destruction:** The implementation object is eligible for garbage collection once it is no longer reachable, following standard Java memory management rules.

## Internal State & Concurrency
- **State:** The interface is inherently stateless. However, an implementation (e.g., a lambda) can be stateful if it is a closure that captures and modifies variables from its enclosing scope.

- **Thread Safety:** The interface itself is thread-safe. The thread safety of a specific *implementation* is entirely the responsibility of the developer who writes it.

    **WARNING:** If a BooleanConsumer implementation modifies shared state (e.g., a field on a surrounding class), the implementation code *must* be synchronized or otherwise made thread-safe if it can be executed by multiple threads concurrently. Failure to do so will result in race conditions.

## API Surface
The public contract consists of a single abstract method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(boolean value) | void | O(N) | Executes the consumer's defined logic with the provided boolean. Complexity is dependent on the implementation. |

## Integration Patterns

### Standard Usage
This interface is intended to be used with lambda expressions for concise, inline implementation of callbacks or event handlers.

```java
// Example: A settings manager using BooleanConsumer for a toggle callback
class GameSettings {
    private boolean vsyncEnabled;
    
    public void onVsyncToggled(BooleanConsumer action) {
        // This action will be executed when the setting changes
        action.accept(this.vsyncEnabled);
    }
}

// Usage
GameSettings settings = new GameSettings();
BooleanConsumer vsyncLogger = (isEnabled) -> {
    System.out.println("V-Sync status is now: " + isEnabled);
};

settings.onVsyncToggled(vsyncLogger);
```

### Anti-Patterns (Do NOT do this)
- **Using `Consumer<Boolean>`:** Avoid using the standard `java.util.function.Consumer<Boolean>` in engine code. It introduces unnecessary object allocation and garbage collector pressure due to boxing the primitive `boolean` into a `Boolean` object. Always prefer the primitive specialization `BooleanConsumer`.
- **Blocking Operations:** Do not implement a BooleanConsumer with long-running or blocking operations (e.g., file I/O, network requests) if it is called on a performance-sensitive thread like the main game loop or rendering thread. This will cause stalls and degrade engine performance.

## Data Pipeline
BooleanConsumer typically acts as a terminal operation or a "sink" in a data flow. It consumes a boolean value and produces a side effect, ending that particular data path.

> Flow:
> Event Source (e.g., UI Button Click) -> Logic Layer (produces boolean result) -> **BooleanConsumer.accept(result)** -> System State Change

