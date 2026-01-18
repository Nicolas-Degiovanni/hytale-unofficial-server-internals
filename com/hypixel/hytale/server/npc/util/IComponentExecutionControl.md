---
description: Architectural reference for IComponentExecutionControl
---

# IComponentExecutionControl

**Package:** com.hypixel.hytale.server.npc.util
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface IComponentExecutionControl {
```

## Architecture & Concepts
The IComponentExecutionControl interface defines a strict contract for managing the execution timing and frequency of server-side NPC behavior components. It acts as a stateful gatekeeper, abstracting away the low-level details of timers, delays, and one-shot triggers from the core behavioral logic.

This abstraction is critical for creating modular and reusable NPC components. Instead of embedding timing logic directly within a component (e.g., an attack or a pathfinding routine), the component delegates the "when to run" decision to an implementation of this interface. This promotes separation of concerns and allows different execution patterns (e.g., "run every 5 seconds", "run once on trigger") to be swapped without altering the component's primary logic. It is a fundamental building block of the server's NPC AI scheduler.

## Lifecycle & Ownership
As an interface, IComponentExecutionControl does not have a lifecycle of its own. Its lifecycle is entirely dictated by the concrete implementation and its owning object, which is typically an NPC behavior component.

-   **Creation:** An implementation of this interface is instantiated alongside its parent NPC component. This usually occurs when an NPC is spawned or when a specific behavior is added to it dynamically.
-   **Scope:** The instance's lifetime is tightly coupled to its owning component. It persists as long as the component is active on the NPC.
-   **Destruction:** The instance is eligible for garbage collection when the parent NPC component is destroyed or removed from the NPC. There are no explicit destruction methods defined in the contract.

## Internal State & Concurrency
-   **State:** Any concrete implementation of this interface is expected to be **mutable**. It must maintain internal state to track time delays (e.g., a countdown timer) and one-shot execution flags (e.g., a boolean `hasFired` flag).
-   **Thread Safety:** This interface is **not thread-safe**. Implementations are designed to be owned and operated by a single NPC entity within the main server game loop. Accessing a single instance from multiple threads will result in race conditions and undefined behavior. All interactions must be synchronized with the server's primary tick thread.

## API Surface
The public contract is minimal, focusing exclusively on time-based and event-based execution gating.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| processDelay(float delta) | boolean | O(1) | Updates an internal timer with the provided time delta. Returns true if the configured delay has elapsed. This is the primary method for time-based throttling. |
| clearOnce() | void | O(1) | Resets the state of a one-shot trigger, allowing it to fire again. |
| setOnce() | void | O(1) | Manually sets a one-shot trigger to its "fired" state, preventing it from running until `clearOnce` is called. |
| isTriggered() | boolean | O(1) | Checks if a one-shot trigger has been fired. This is used for event-based, non-repeating logic. |

## Integration Patterns

### Standard Usage
The intended pattern is for an NPC component to hold a private instance of an IComponentExecutionControl implementation. During the component's update tick, it calls a method on the control object to determine if its core logic should execute.

```java
// Example within a hypothetical NPC component's update method
// this.executionControl is an IComponentExecutionControl implementation

public void onTick(float deltaTime) {
    // This component's logic will only run once every 2.5 seconds
    if (this.executionControl.processDelay(deltaTime)) {
        // Execute core component logic, e.g., scan for targets
        performTargetScan();
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Shared Instances:** Never share a single IComponentExecutionControl instance between multiple, independent NPC components. This will cause severe logic conflicts as components overwrite each other's timer and trigger states.
-   **External State Management:** Do not manage timing or trigger flags outside of the control object. The purpose of this interface is to encapsulate that state. Bypassing it defeats the design.
-   **Ignoring Delta Time:** Calling `processDelay` with a static value instead of the actual frame delta time will break the timing mechanism, making it frame-rate dependent and unreliable.

## Data Pipeline

This component functions as a control-flow gate rather than a data-processing pipeline. Its role is to conditionally halt or permit the execution of subsequent logic.

> Flow:
> Server Tick -> NPC Update -> Component Tick -> **IComponentExecutionControl.processDelay()** -> [Execution Allowed/Blocked] -> Component's Core Logic

