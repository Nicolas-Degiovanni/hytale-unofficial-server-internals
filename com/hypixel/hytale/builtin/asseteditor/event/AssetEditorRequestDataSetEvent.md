---
description: Architectural reference for AssetEditorRequestDataSetEvent
---

# AssetEditorRequestDataSetEvent

**Package:** com.hypixel.hytale.builtin.asseteditor.event
**Type:** Transient

## Definition
```java
// Signature
public class AssetEditorRequestDataSetEvent implements IAsyncEvent<String> {
```

## Architecture & Concepts
The AssetEditorRequestDataSetEvent class is a specialized Data Transfer Object (DTO) that represents a request for a collection of data from a connected Asset Editor client. It functions as a message within the engine's event bus system, serving to decouple the network layer from the data-providing systems.

By implementing the IAsyncEvent interface, this event signals to the event bus that its handlers can be executed on a worker thread. This is a critical performance feature, as it prevents the network or main thread from blocking while potentially expensive data lookups (e.g., file system scans or database queries) are performed.

The core architectural pattern here is **Request-Reply via an Asynchronous Event**. The event is created to encapsulate a request, dispatched, and then mutated by a handler to contain the reply. The system that initiated the request can then await the completion of the event to retrieve the results.

## Lifecycle & Ownership
- **Creation:** Instantiated by the EditorClient or a related network protocol handler when a message arrives from an external Asset Editor tool requesting a specific named data set.
- **Scope:** The object's lifetime is brief and tied directly to the handling of a single request. It exists only for the duration of its travel through the event bus and processing by its designated handler.
- **Destruction:** Once all handlers have completed and the asynchronous operation is finalized, the event object has no remaining references and becomes eligible for garbage collection. There is no manual destruction mechanism.

## Internal State & Concurrency
- **State:** This object is intentionally mutable. While the request context (editorClient, dataSet) is immutable and set at creation via final fields, the *results* field is designed to be populated by a downstream event handler. The object transitions from a "request" state to a "response" state during its lifecycle.
- **Thread Safety:** This class is **not thread-safe** and must not be treated as such. The design relies on the event bus to guarantee safe publication to a single handler thread. Concurrent calls to setResults from multiple threads will result in a race condition and undefined behavior. The expected concurrency model is a single producer (the network thread) and a single consumer (the handler thread).

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDataSet() | String | O(1) | Returns the unique string identifier for the requested data set. |
| getEditorClient() | EditorClient | O(1) | Returns the client session instance that initiated this request. |
| getResults() | String[] | O(1) | Retrieves the results populated by the event handler. May be null if the event has not yet been handled. |
| setResults(String[] results) | void | O(1) | Populates the event with the requested data. This is the primary mutation point for event handlers. |

## Integration Patterns

### Standard Usage
The primary integration pattern is to subscribe a handler to this event. The handler is responsible for interpreting the request, fetching the appropriate data, and populating the event object with the results.

```java
// Example of a handler in a data-providing service
@Subscribe
public void onDataSetRequest(AssetEditorRequestDataSetEvent event) {
    // Avoid blocking by submitting to a dedicated service if the lookup is slow
    String[] data = assetIndex.queryByName(event.getDataSet());

    // Mutate the event object to contain the reply
    event.setResults(data);

    // The event bus framework handles the completion signal. The originator
    // of the event will now be ableto access the results.
}
```

### Anti-Patterns (Do NOT do this)
- **Reusing Event Instances:** Do not cache and re-dispatch an instance of this event. Each request from the Asset Editor must result in a new AssetEditorRequestDataSetEvent object.
- **Creator Populates Results:** The creator of the event should not populate the results field. The purpose of the event is to *request* data, not to distribute pre-existing data. The results field should be set to null or an empty array upon construction.
- **Multiple Handlers Modifying State:** Do not register multiple event handlers that all attempt to call setResults on the same event. This creates a race condition where the last handler to execute will overwrite all previous results.

## Data Pipeline
The event acts as a carrier in a well-defined data flow between the network layer and backend services.

> Flow:
> Network Message (from external tool) -> EditorClient (deserializes) -> **AssetEditorRequestDataSetEvent** (created and fired) -> Event Bus (dispatches to worker thread) -> Data Service Handler (processes request) -> `setResults()` is called -> Event Bus (signals completion) -> EditorClient (retrieves results and sends network response)

