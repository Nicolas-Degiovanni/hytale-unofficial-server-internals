---
description: Architectural reference for EventPriority
---

# EventPriority

**Package:** com.hypixel.hytale.event
**Type:** Utility

## Definition
```java
// Signature
public enum EventPriority {
```

## Architecture & Concepts
EventPriority is a foundational component of the engine's event bus system. It provides a strongly-typed, ordered set of constants that dictate the execution order of event listeners (subscribers). This enumeration is critical for establishing a deterministic and predictable event handling pipeline, preventing race conditions and ensuring that critical systems can process events before or after dependent systems.

The primary architectural role of EventPriority is to decouple event listeners from each other while still enforcing a strict sequence of operations. For example, a networking system might need to handle a packet event with FIRST priority to update the world state, while a UI system listens with LATE priority to reflect that change on the screen only after it is finalized.

The underlying short values are not arbitrary. They are spaced far apart, allowing developers to potentially define intermediate custom priorities for plugins or mods without conflicting with the core engine's enumeration values.

## Lifecycle & Ownership
- **Creation:** As a Java enum, all constants (FIRST, EARLY, NORMAL, LATE, LAST) are instantiated by the Java Virtual Machine during class loading. This process is managed entirely by the JVM and occurs once when the EventPriority class is first referenced.
- **Scope:** The enum constants are static, final, and globally accessible. They persist for the entire lifetime of the application.
- **Destruction:** The constants are garbage collected only when the application's ClassLoader is unloaded, which typically happens only at application shutdown.

## Internal State & Concurrency
- **State:** EventPriority is immutable. Each enum constant holds a single, final short field named value, which is assigned at compile time and cannot be changed.
- **Thread Safety:** This class is inherently thread-safe. Its immutability guarantees that it can be safely accessed and read from any thread without synchronization. The values are compile-time constants, eliminating any possibility of concurrent modification.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | short | O(1) | Returns the underlying short value associated with the priority constant. |

## Integration Patterns

### Standard Usage
EventPriority is not used directly but as a parameter within event subscription annotations. The EventBus reads this metadata via reflection to build its sorted invocation list for each event type.

```java
// A listener method in a service class
// This method will be called after NORMAL priority listeners.
@Subscribe(priority = EventPriority.LATE)
public void onPlayerJoin(PlayerJoinEvent event) {
    // Update UI elements after game state is settled
    uiManager.showWelcomeMessage(event.getPlayer());
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Comparison:** Do not compare priorities using their raw short values. This breaks type safety and makes the code harder to read and maintain. Rely on the enum's natural ordering or identity.

```java
// BAD: Magic number comparison
if (listener.getPriority().getValue() > 0) {
    // ...
}

// GOOD: Rely on enum properties
if (listener.getPriority() == EventPriority.LATE) {
    // ...
}
```

- **Extending the Enum:** It is a language-level constraint that enums cannot be extended. Do not attempt to create a subclass of EventPriority.

## Data Pipeline
EventPriority does not process data. Instead, it acts as metadata that governs the control flow of the event dispatch pipeline. The EventBus uses the priority to sort the list of subscribers before an event is dispatched to them.

> Flow:
> Event Dispatched -> EventBus retrieves all subscribers for the event type -> **Subscribers are sorted using EventPriority** -> Subscriber (FIRST) -> Subscriber (EARLY) -> Subscriber (NORMAL) -> Subscriber (LATE) -> Subscriber (LAST)

