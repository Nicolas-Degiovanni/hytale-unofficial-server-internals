---
description: Architectural reference for TickingThread
---

# TickingThread

**Package:** com.hypixel.hytale.server.core.util.thread
**Type:** Base Class

## Definition
```java
// Signature
public abstract class TickingThread implements Runnable {
```

## Architecture & Concepts
The TickingThread is an abstract base class that provides the core implementation for a fixed-rate, looping thread. It is a fundamental construct in the server architecture, designed to be the "heartbeat" for any system that requires periodic execution at a consistent frequency, measured in Ticks Per Second (TPS).

This class abstracts away the complex and error-prone logic of timing, thread sleeping, and spin-waiting required to maintain a stable TPS. Subclasses are only required to implement the `tick` method, which contains the specific logic to be executed each cycle. This pattern is used extensively for major systems like the main game loop, world processing, and potentially dedicated network or AI threads.

By centralizing the loop management logic, TickingThread ensures that all major server subsystems operate on a predictable and synchronized schedule, which is critical for simulation consistency and performance monitoring.

### Lifecycle & Ownership
- **Creation:** A concrete subclass of TickingThread is instantiated by a high-level manager or application entry point. For example, a `WorldManager` would create a `WorldTickingThread` for each active game world. The abstract TickingThread class itself is never instantiated directly.
- **Scope:** The lifetime of a TickingThread instance is coupled to the component it serves. A main server loop thread will persist for the entire application session. A world-specific thread will live only as long as that world is loaded and active.
- **Destruction:** The thread is terminated explicitly by an owner thread calling `stop()` or `interrupt()`. The `stop` method attempts a graceful shutdown by interrupting the loop and joining the thread, with a timeout to prevent deadlocks. If the thread fails to terminate gracefully, it is forcibly stopped. The `onShutdown` hook provides a guaranteed final opportunity for subclasses to clean up resources.

## Internal State & Concurrency
- **State:** The class maintains a highly mutable internal state, including the target `tps`, the calculated `tickStepNanos`, a reference to its managed `Thread`, and atomic flags like `needsShutdown` for cross-thread communication. It also contains a `HistoricMetric` object to collect performance data about tick execution times.
- **Thread Safety:** This class is **not thread-safe** and is designed for a single-threaded execution model. The core `run` loop and the `tick` method it invokes are confined to the internal thread managed by the instance.

    **WARNING:** Methods that modify internal state, such as `setTps`, are explicitly designed to be called *only from within the ticking thread itself*. An assertion, `debugAssertInTickingThread`, enforces this rule in development environments. Lifecycle methods like `start` and `stop` are the only methods safe to call from an external, managing thread.

## API Surface
The public and protected API provides the contract for lifecycle management and subclass implementation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| start() | CompletableFuture<Void> | O(1) | Creates and starts the internal thread. Returns a future that completes when the thread's `onStart` method finishes. Throws IllegalStateException if already started. |
| stop() | void | O(N) | Attempts a graceful shutdown with a 30-second timeout. Manages thread interruption and joining. **WARNING:** This method can block the calling thread. |
| interrupt() | boolean | O(1) | Sends an interrupt signal to the internal thread, breaking the main loop. |
| tick(float delta) | protected abstract void | User-defined | Abstract method to be implemented by subclasses. Contains the logic to be executed each tick. |
| onStart() | protected void | User-defined | A hook method called once at the beginning of the thread's execution, before the main loop begins. |
| onShutdown() | protected abstract void | User-defined | An abstract hook method called once when the thread is terminating, either gracefully or forcefully. Guaranteed to be called for resource cleanup. |
| isInThread() | boolean | O(1) | Returns true if the current code is executing on this TickingThread's managed thread. |

## Integration Patterns

### Standard Usage
A developer must extend TickingThread and implement the abstract methods. The lifecycle is then managed externally.

```java
// 1. Create a concrete implementation
class GameLoopThread extends TickingThread {
    public GameLoopThread() {
        super("GameLoop", 30, false); // Name, TPS, isDaemon
    }

    @Override
    protected void onStart() {
        System.out.println("Game loop is starting...");
    }

    @Override
    protected void tick(float delta) {
        // Execute one cycle of game logic, physics, etc.
        World.update(delta);
    }

    @Override
    protected void onShutdown() {
        System.out.println("Game loop is shutting down, saving state...");
    }
}

// 2. Manage its lifecycle from an owner context
GameLoopThread gameLoop = new GameLoopThread();
gameLoop.start().join(); // Start the thread and wait for initialization

// ... later, during server shutdown
gameLoop.stop();
```

### Anti-Patterns (Do NOT do this)
- **Long-Running Ticks:** Implementing a `tick` method that performs long-running or blocking operations. This will prevent the thread from maintaining its target TPS, causing severe performance degradation and simulation instability. Each tick must complete well within the `tickStepNanos` budget.
- **Cross-Thread State Mutation:** Calling methods like `setTps` from a thread other than the TickingThread itself. This violates the class's concurrency model and will trigger an `AssertionError`.
- **Ignoring Shutdown:** Failing to call `stop` during application shutdown. This will result in a leaked thread resource that may prevent the JVM from exiting cleanly.

## Data Pipeline
TickingThread does not process a flow of data in a traditional sense. Instead, it acts as a **time-based event generator**. It originates the "pulse" that drives other data processing systems.

> Flow:
> System Clock (nanotime) -> **TickingThread Run Loop** -> `tick()` Invocation -> Subsystem Logic (e.g., World Simulation, AI Processing, Network Buffer Flush)

