---
description: Architectural reference for ValueWrappedInfoProvider
---

# ValueWrappedInfoProvider

**Package:** com.hypixel.hytale.server.npc.sensorinfo
**Type:** Transient

## Definition
```java
// Signature
public class ValueWrappedInfoProvider implements InfoProvider {
```

## Architecture & Concepts
The ValueWrappedInfoProvider is a structural component within the server-side NPC sensory system. It implements the **Decorator** pattern to construct a chain of responsibility for resolving sensory data. This class is not a standalone provider; its primary function is to wrap an existing InfoProvider, augmenting it with a specific ParameterProvider.

This design creates a composable, linked-list-style structure where each node in the chain can override or provide new sensory parameters. When an NPC's AI requests a parameter via getParameterProvider, the request traverses the chain. The first ValueWrappedInfoProvider that can resolve the parameter with its own internal ParameterProvider will do so and terminate the search. If it cannot, it delegates the request to the next InfoProvider it wraps.

All other requests, such as for position or extra info, are passed directly down the chain without modification. This architecture allows for dynamic and context-sensitive sensory information. For example, an NPC's current behavior (e.g., "in combat") can prepend a ValueWrappedInfoProvider to the chain to temporarily override its default sensory parameters without altering the base configuration.

## Lifecycle & Ownership
- **Creation:** Instantiated dynamically by higher-level AI systems, such as a Behavior Tree or State Machine, when constructing a sensory context for an NPC. It is common to see a new chain of these providers built for a single AI evaluation cycle.
- **Scope:** Extremely short-lived. A ValueWrappedInfoProvider instance typically exists only for the duration of a single sensory check or AI logic update within a game tick.
- **Destruction:** The object is eligible for garbage collection as soon as the AI system that created it discards its reference to the head of the provider chain. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The class is effectively **immutable**. Its internal fields, wrappedProvider and parameterProvider, are final and assigned only at construction. The state of the objects it references may be mutable, but this wrapper itself does not change.
- **Thread Safety:** This class is **conditionally thread-safe**. It contains no internal locks or mutable state. Its safety in a multi-threaded environment is entirely dependent on the thread safety of the InfoProvider and ParameterProvider instances it holds. In its intended use within the single-threaded NPC update loop, concurrency is not a primary concern.

**WARNING:** Using this class with non-thread-safe providers in an asynchronous or multi-threaded AI system will lead to unpredictable behavior and race conditions.

## API Surface
The public API is an implementation of the InfoProvider interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPositionProvider() | IPositionProvider | O(N) | Delegates the call down the entire wrapped provider chain. |
| getParameterProvider(int) | ParameterProvider | O(N) | Attempts to resolve with its local provider; otherwise, delegates down the chain. |
| getExtraInfo(Class) | E | O(N) | Delegates the call down the entire wrapped provider chain. |
| passExtraInfo(E) | void | O(N) | Passes an ExtraInfoProvider down the entire wrapped provider chain. |
| getPassedExtraInfo(Class) | E | O(N) | Delegates the call down the entire wrapped provider chain. |
| hasPosition() | boolean | O(N) | Delegates the call down the entire wrapped provider chain. |

*Complexity Note: N represents the number of providers in the wrapped chain.*

## Integration Patterns

### Standard Usage
The primary pattern is to build a chain of providers, where each new provider wraps the previous one to add or override specific parameters.

```java
// 1. Start with a base provider (e.g., from the NPC entity itself)
InfoProvider baseProvider = npc.getSensing().getInfoProvider();

// 2. Wrap it to add context-specific parameters for a "fleeing" state
ParameterProvider fleeingParams = new FleeingParameterProvider(target);
InfoProvider fleeingContext = new ValueWrappedInfoProvider(baseProvider, fleeingParams);

// 3. The AI system now uses the new head of the chain
npc.getBehaviorTree().evaluate(fleeingContext);
```

### Anti-Patterns (Do NOT do this)
- **Circular Wrapping:** Never create a situation where a provider wraps another provider that, in turn, wraps the original. This will cause a StackOverflowError on any method call due to infinite recursion.
- **Stateful Delegation:** Do not design a ParameterProvider for use in this class that relies on mutable state modified by other threads. The lookup chain is not guaranteed to be atomic.

## Data Pipeline
The data flow for this component is a directed, one-way traversal for information requests. It acts as a conditional gate in a pipeline.

> Flow for getParameterProvider:
> AI Sensor Request -> **ValueWrappedInfoProvider.getParameterProvider()** -> Check local ParameterProvider -> (Success) -> Return Value
>
> (Failure) -> Delegate to **wrappedProvider.getParameterProvider()** -> ... -> Base Provider -> Return Value or Null

