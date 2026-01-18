---
description: Architectural reference for EventData
---

# EventData

**Package:** com.hypixel.hytale.server.core.ui.builder
**Type:** Value Object

## Definition
```java
// Signature
public record EventData(Map<String, String> events) {
```

## Architecture & Concepts
EventData is a specialized, server-side data structure designed to construct key-value payloads for UI events. It acts as a lightweight wrapper around a Map, providing a fluent builder-style API for ease of use.

Architecturally, it is implemented as a Java record, signaling its primary intent as a simple, transparent data carrier. However, its design presents a notable contradiction: while records are typically immutable, EventData exposes methods like *put* and *append* that directly mutate its internal map. This pattern is known as **shallow immutability**; the reference to the map within the record is final, but the map itself is mutable.

This design choice prioritizes the convenience of a builder pattern over strict immutability. The use of fastutil's Object2ObjectOpenHashMap in the default constructor indicates a performance-conscious decision, aiming to reduce memory overhead and improve access times for the underlying map structure.

## Lifecycle & Ownership
- **Creation:** EventData instances are ephemeral and created on-demand. They are typically instantiated via the no-argument constructor at the beginning of a UI component's event-building process. The static factory *of* provides a convenience for creating an instance with a single, pre-defined entry.
- **Scope:** The lifecycle of an EventData object is extremely short. It exists only for the duration of constructing a single UI element or event payload. It is a transient object, not intended to be stored or reused.
- **Destruction:** Once the containing UI component is serialized or the event is dispatched, the EventData object is no longer referenced and becomes eligible for garbage collection.

## Internal State & Concurrency
- **State:** The component's state is the mutable Map of string key-value pairs it encapsulates. The state grows as developers chain *append* or *put* calls. The shallow immutability of the record means that any system holding a reference to an EventData instance can alter its internal state.

- **Thread Safety:** **This class is not thread-safe.** The underlying Object2ObjectOpenHashMap is not synchronized. Accessing and modifying a single EventData instance from multiple threads without external locking mechanisms will result in race conditions and unpredictable behavior. It is designed exclusively for single-threaded construction within the scope of a single operation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| EventData() | constructor | O(1) | Creates an empty EventData container, backed by a high-performance map. |
| append(key, value) | EventData | O(1) amortized | Fluent API method to add a key-value pair. Returns the same instance for chaining. |
| append(key, enumValue) | EventData | O(1) amortized | Fluent API convenience method to add an enum's name as the value. |
| put(key, value) | EventData | O(1) amortized | The core mutation method. Adds or overwrites a key-value pair in the internal map. |
| of(key, value) | static EventData | O(1) | Static factory to create a new EventData instance populated with a single entry. |

## Integration Patterns

### Standard Usage
EventData is intended to be used as a chained builder to construct a set of parameters that will be attached to a UI event.

```java
// How a developer should normally use this
EventData clickEvent = new EventData()
    .append("action", "OPEN_INVENTORY")
    .append("target", "player_main_backpack");

// The resulting 'clickEvent' object is then passed to a UI component builder
// which will serialize it for the client.
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Never share an EventData instance between threads. The internal state is not protected against concurrent writes.

    ```java
    // DANGEROUS: Do not do this
    EventData sharedEvent = new EventData();
    new Thread(() -> sharedEvent.append("source", "thread_A")).start();
    new Thread(() -> sharedEvent.append("source", "thread_B")).start();
    // The final state of sharedEvent.events is unpredictable.
    ```

- **Treating as Immutable:** Do not pass an EventData instance to other systems with the expectation that it is a read-only value object. Any method can modify its contents.

    ```java
    // WARNING: Potential for unexpected side-effects
    EventData event = new EventData().append("id", "123");
    someOtherSystem.process(event); // This method might add or remove keys
    ```

## Data Pipeline
EventData serves as the initial collection point for data that will eventually be consumed by the client's UI system. It does not process data itself; it is a temporary container.

> Flow:
> Server-Side Logic -> **new EventData()** -> Chained `append()` calls -> UI Component Builder -> Game Object Serializer -> Network Packet -> Client UI Engine

