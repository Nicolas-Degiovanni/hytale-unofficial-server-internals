---
description: Architectural reference for RunWhenPausedSystem
---

# RunWhenPausedSystem

**Package:** com.hypixel.hytale.component.system.tick
**Type:** Contract Interface

## Definition
```java
// Signature
public interface RunWhenPausedSystem<ECS_TYPE> extends TickableSystem<ECS_TYPE> {
    // This interface adds no new methods.
}
```

## Architecture & Concepts
The RunWhenPausedSystem interface is a **marker interface** used by the engine's core ticking mechanism. Its sole purpose is to signal to the SystemTicker that a system should continue to receive tick updates even when the primary game simulation is in a paused state.

This contract is fundamental to maintaining a responsive user experience during pauses. While core gameplay systems like physics, AI, and entity updates are halted, systems responsible for UI animations, background audio, or certain network keep-alives must continue to operate. By implementing this interface, a system opts into the "paused tick" loop, effectively separating itself from the main game simulation loop.

This is a classic example of an **Inversion of Control** pattern. Instead of a system querying the game state to see if it should run, the SystemTicker uses this type information to control the system's execution, decoupling the system from the global pause state management.

### Lifecycle & Ownership
As an interface, RunWhenPausedSystem itself has no lifecycle. The lifecycle described below applies to the concrete classes that **implement** this interface.

- **Creation:** Concrete system instances are typically created by a dependency injection framework or a service locator during the client or server bootstrap sequence. They are registered with the central SystemTicker.
- **Scope:** An implementing system's lifetime is tied to the scope in which it was registered. For most UI or client-level systems, this means they persist for the entire client session.
- **Destruction:** Systems are destroyed when their parent context is torn down, such as when shutting down the client or unloading a world.

## Internal State & Concurrency
- **State:** This interface is stateless. Any state is managed by the implementing class. Implementers should be designed with the awareness that they operate on a potentially frozen or static game state.
- **Thread Safety:** The contract makes no guarantees about thread safety. The SystemTicker will invoke methods on implementing systems from a specific, predictable thread (e.g., the main client thread). It is the responsibility of the implementing class to ensure its own thread safety if it interacts with asynchronous services.

## API Surface
This interface introduces no new methods. It inherits its entire API surface from the parent interface, TickableSystem. Its value is derived from its type, not its contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(ECS_TYPE) | void | Varies | Inherited from TickableSystem. Executes one frame of logic for the system. |

## Integration Patterns

### Standard Usage
A developer should implement this interface on any system that must remain active when the game is paused. This is common for UI, audio, and some networking systems.

```java
// A system that animates a menu should continue running when paused.
public class MenuAnimationSystem implements RunWhenPausedSystem<ClientECS> {

    @Override
    public void tick(ClientECS ecs) {
        // Update UI element positions, fade effects, etc.
        // This logic will execute even if the game world is frozen.
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Modifying Game State:** Do not implement this interface on systems that modify core gameplay state (e.g., entity positions, health, or physics). Doing so will cause the game simulation to advance while the user perceives it as paused, leading to desynchronization, bugs, and exploits.
- **Heavy Computation:** Systems that run when paused should be lightweight. Avoid performing heavy computations, as this can degrade performance in menus and loading screens where a smooth frame rate is expected.

## Data Pipeline
RunWhenPausedSystem does not process data. Instead, it alters the **control flow** within the SystemTicker.

> Flow:
> SystemTicker Loop -> Check Global `isPaused` Flag
>
> **If Paused:**
> Iterate Systems -> Check `system instanceof RunWhenPausedSystem` -> If true, invoke `system.tick()`
>
> **If Not Paused:**
> Iterate Systems -> Invoke `system.tick()` on all registered systems

