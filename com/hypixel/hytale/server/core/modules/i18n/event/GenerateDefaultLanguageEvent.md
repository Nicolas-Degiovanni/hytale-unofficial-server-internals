---
description: Architectural reference for GenerateDefaultLanguageEvent
---

# GenerateDefaultLanguageEvent

**Package:** com.hypixel.hytale.server.core.modules.i18n.event
**Type:** Transient

## Definition
```java
// Signature
public class GenerateDefaultLanguageEvent implements IEvent<Void> {
```

## Architecture & Concepts
The GenerateDefaultLanguageEvent is a message-oriented data transfer object used within the server's core event bus. It is not a service or manager, but rather an ephemeral container that facilitates a decoupled data aggregation workflow.

Its primary architectural role is to orchestrate the collection of default (en-us) translation strings from disparate modules during server initialization or asset reloading. A central system fires this event, and various listeners, each responsible for a specific feature or content pack, respond by populating a shared translation map.

This design effectively inverts control; the central internationalization (i18n) system does not need to know about every possible source of translations. Instead, modules can independently register as listeners and contribute their data when this event is dispatched. The implementation of IEvent with a Void type parameter signifies that this is a one-way notification; handlers are not expected to return a result.

## Lifecycle & Ownership
- **Creation:** Instantiated by a high-level orchestrator, typically the I18nModule or a server bootstrap manager, at the beginning of the language file generation process. The creator is also responsible for instantiating the ConcurrentHashMap that this event will carry.
- **Scope:** Extremely short-lived. An instance of this event exists only for the duration of its dispatch through the event bus. Once all listeners have processed it, it is no longer referenced and becomes eligible for garbage collection.
- **Destruction:** The event object is destroyed by the garbage collector shortly after the event dispatch completes. **Warning:** The ConcurrentHashMap that it references is *not* owned by the event and will persist long after the event is gone.

## Internal State & Concurrency
- **State:** The internal state consists of a single, final reference to a ConcurrentHashMap. While the reference itself is immutable, the map it points to is highly mutable and is expected to be modified by multiple event handlers. This object acts as a temporary, thread-safe handle to a shared, mutable state.
- **Thread Safety:** This class is inherently thread-safe. The core operation, putTranslationFile, delegates directly to ConcurrentHashMap.put, which is a lock-free, thread-safe operation. This is a critical design feature, as it allows different modules or asset loaders operating on separate threads to contribute translations concurrently without data corruption or the need for external synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| putTranslationFile(filename, translations) | void | O(1) avg. | Adds a complete set of translations for a given file to the shared map. This is the primary mechanism for event handlers to contribute data. |

## Integration Patterns

### Standard Usage
The event is intended to be used in a "scatter-gather" pattern. An orchestrator creates the shared map and the event, dispatches it, and then, after the dispatch is complete, processes the now-populated map.

```java
// How the i18n system should use this
ConcurrentHashMap<String, TranslationMap> sharedMap = new ConcurrentHashMap<>();
GenerateDefaultLanguageEvent event = new GenerateDefaultLanguageEvent(sharedMap);

// The event bus will deliver this to all interested listeners
eventBus.fire(event);

// After firing, the sharedMap is now populated by all listeners
processAndWriteLanguageFiles(sharedMap);
```

### Anti-Patterns (Do NOT do this)
- **Holding a Reference:** Do not store a reference to this event object in a field or long-lived variable. Its lifecycle is intentionally ephemeral.
- **Instantiation by Listeners:** Event handlers must only *consume* this event. They must never create a new instance of GenerateDefaultLanguageEvent.
- **Map Manipulation:** Handlers should only perform additive operations via putTranslationFile. Attempting to clear the map or remove entries from other modules can lead to unpredictable system behavior and data loss.

## Data Pipeline
This event facilitates a data aggregation pipeline. It acts as the vehicle for collecting data fragments from multiple sources into a single, unified data structure.

> Flow:
> I18nModule creates ConcurrentHashMap -> I18nModule creates **GenerateDefaultLanguageEvent** -> Event Bus dispatches to listeners -> Listeners populate map via event -> I18nModule processes populated map -> Default language file is generated on disk

