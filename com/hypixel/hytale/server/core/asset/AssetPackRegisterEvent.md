---
description: Architectural reference for AssetPackRegisterEvent
---

# AssetPackRegisterEvent

**Package:** com.hypixel.hytale.server.core.asset
**Type:** Transient

## Definition
```java
// Signature
public class AssetPackRegisterEvent implements IEvent<Void> {
```

## Architecture & Concepts
The AssetPackRegisterEvent is a message-oriented data structure that functions as a core component of the server's event-driven asset management system. Its primary architectural role is to decouple the systems responsible for *loading* asset packs from the systems responsible for *registering and indexing* them.

When a component, such as a file system monitor or a content downloader, successfully loads an AssetPack, it does not directly interact with the central AssetManager. Instead, it constructs and dispatches an AssetPackRegisterEvent onto the server's main event bus. This pattern promotes a highly modular and extensible system where new sources of assets can be added without modifying the core registration logic.

The implementation of IEvent with a Void generic type indicates that this is a "fire-and-forget" notification. Systems that dispatch this event do not expect a direct return value or confirmation from the listeners.

## Lifecycle & Ownership
- **Creation:** Instantiated by any system that discovers or loads a new AssetPack. This typically occurs during server startup when loading base assets or dynamically when a new content pack is introduced.
- **Scope:** Extremely short-lived. An instance of this event exists only for the duration of its propagation through the event bus. It is a transient data carrier, not a long-lived service.
- **Destruction:** The object becomes eligible for garbage collection immediately after all subscribed event listeners have processed it. There is no manual destruction or cleanup required.

## Internal State & Concurrency
- **State:** Immutable. The internal AssetPack reference is declared as final and is assigned only once during construction. This guarantees that the event's payload cannot be altered after it has been dispatched.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. It can be safely published and consumed across different threads without requiring locks or other synchronization primitives.

    **WARNING:** While the event object itself is thread-safe, the contained AssetPack object may not be. The architectural contract implies that the AssetPack is in a read-only state by the time it is wrapped in this event. Modifying the AssetPack from a listener constitutes a severe anti-pattern.

## API Surface
The public contract is minimal, focused solely on data retrieval.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetPack() | AssetPack | O(1) | Returns the non-null AssetPack payload intended for registration. |

## Integration Patterns

### Standard Usage
The correct pattern is to create the event and immediately post it to the application's central event bus. Subscribing systems, such as the AssetManager, will listen for this specific event type.

```java
// Example: A system loads an AssetPack from a file
AssetPack loadedPack = AssetPackLoader.loadFromFile("path/to/pack");

// Dispatch the event for other systems to consume
// The eventBus is typically injected or retrieved from a service context.
eventBus.post(new AssetPackRegisterEvent(loadedPack));
```

### Anti-Patterns (Do NOT do this)
- **State Mutation:** Do not retrieve the AssetPack from the event and then attempt to modify its contents. The event signifies that an asset pack is *ready for registration*, not that it is open for modification.
- **Direct Coupling:** Avoid creating the event and passing it directly to a manager. This circumvents the event bus and creates a tight coupling between the asset loader and the asset registry, defeating the architectural purpose of the event.

    ```java
    // BAD: This creates a direct dependency and bypasses other potential listeners.
    AssetPack pack = loadMyPack();
    AssetPackRegisterEvent event = new AssetPackRegisterEvent(pack);
    assetManager.manuallyHandleRegistration(event); // Defeats the purpose of the event bus
    ```

## Data Pipeline
The AssetPackRegisterEvent acts as the data carrier in the asset registration pipeline. It represents a single, discrete unit of work being passed from a producer to one or more consumers.

> Flow:
> AssetPack Loader -> **AssetPackRegisterEvent (Creation)** -> Event Bus (Dispatch) -> AssetManager (Consumption) -> Resource Index Update

