---
description: Architectural reference for UIEventBuilder
---

# UIEventBuilder

**Package:** com.hypixel.hytale.server.core.ui.builder
**Type:** Transient

## Definition
```java
// Signature
public class UIEventBuilder {
```

## Architecture & Concepts
The UIEventBuilder is a server-side utility that implements the Builder design pattern. Its primary function is to construct a collection of event bindings for custom user interfaces before they are sent to the client. This class acts as a high-level abstraction, decoupling the game logic that defines UI interactivity from the low-level serialization and packet construction process.

Developers use this builder to define what happens when a player interacts with a UI element. For example, it can bind a "click" event on a button (identified by a selector) to a specific server-side action. The builder aggregates these bindings and serializes any associated metadata into a JSON format suitable for network transmission. The final output is an array of CustomUIEventBinding objects, a data structure ready for inclusion in a larger UI definition packet.

## Lifecycle & Ownership
- **Creation:** An instance is created directly via its constructor, for example `new UIEventBuilder()`. It is typically instantiated within a method responsible for generating a specific UI screen or component.
- **Scope:** The object is short-lived and method-scoped. Its existence is tied to the single operation of building one set of UI events. It is not designed to be stored, cached, or reused.
- **Destruction:** The instance becomes eligible for garbage collection as soon as the `getEvents` method is called and its result is passed to the consuming system, typically a packet constructor.

## Internal State & Concurrency
- **State:** The UIEventBuilder is a stateful, mutable object. It maintains an internal `ObjectArrayList` of CustomUIEventBinding objects, which grows with each call to `addEventBinding`.

- **Thread Safety:** **This class is not thread-safe and must not be shared across threads.** Its internal list is not synchronized. More critically, it relies on a `ThreadLocal` variable (ExtraInfo.THREAD_LOCAL) for the serialization context. Concurrent access will lead to race conditions, data corruption, and unpredictable serialization errors. An instance must only be used by the thread that created it.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addEventBinding(type, selector, data, locksInterface) | UIEventBuilder | O(N) | Adds a new event binding. Serializes the provided EventData into a JSON string. N is the size of the data to be serialized. Returns the builder instance for chaining. |
| getEvents() | CustomUIEventBinding[] | O(M) | Finalizes the build process and returns the complete collection of event bindings as a new array. M is the number of events added. |

## Integration Patterns

### Standard Usage
The builder is designed for a fluent, chained invocation style. A new instance is created, configured with all necessary event bindings, and then finalized by calling `getEvents`.

```java
// How a developer should normally use this
// Assume 'createMyUIPacket' is a method that needs the event bindings.

CustomUIEventBinding[] bindings = new UIEventBuilder()
    .addEventBinding(CustomUIEventBindingType.CLICK, "#confirm_button")
    .addEventBinding(CustomUIEventBindingType.HOVER, "#info_icon", createTooltipData(), false)
    .getEvents();

createMyUIPacket(bindings);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not hold a reference to a UIEventBuilder after calling `getEvents` with the intent to add more events. The internal state is not cleared, which will result in duplicate or incorrect event lists. Always create a new instance for each distinct UI definition.
- **Concurrent Modification:** Do not pass a UIEventBuilder instance to another thread or access it concurrently. This is explicitly unsupported and will break the `ThreadLocal` serialization context.
- **Storing as a Field:** Do not store a UIEventBuilder as a field in a long-lived service or component. Its lifecycle is meant to be transient.

## Data Pipeline
The UIEventBuilder serves as a critical step in the UI serialization pipeline, transforming high-level server-side objects into a network-ready format.

> Flow:
> High-Level EventData Object -> MapCodec Serialization -> **UIEventBuilder** -> CustomUIEventBinding[] -> UI Definition Packet -> Network Layer

