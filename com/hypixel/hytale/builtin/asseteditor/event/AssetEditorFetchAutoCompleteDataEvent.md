---
description: Architectural reference for AssetEditorFetchAutoCompleteDataEvent
---

# AssetEditorFetchAutoCompleteDataEvent

**Package:** com.hypixel.hytale.builtin.asseteditor.event
**Type:** Transient

## Definition
```java
// Signature
public class AssetEditorFetchAutoCompleteDataEvent implements IAsyncEvent<String> {
```

## Architecture & Concepts
The AssetEditorFetchAutoCompleteDataEvent is a message-oriented data structure that facilitates asynchronous data lookups for the in-game Asset Editor UI. It serves as a formal request contract between a UI component (e.g., a text input field) and a background data provider.

By implementing the IAsyncEvent interface, this class signals to the event system that its processing may occur off the main thread, preventing UI freezes during potentially slow data queries. The event encapsulates all necessary context for a handler to perform an autocomplete search: the client context, the specific category of data being searched (the data set), and the user's partial input (the query).

Its primary architectural role is to decouple the user interface from the data-access layer. The UI does not need to know which service provides autocomplete data; it simply fires this event, and a registered listener fulfills the request by populating the event object itself.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand by a UI component within the Asset Editor, typically in response to a user typing into a text field that supports autocomplete.
- **Scope:** This object has a very short lifecycle. It exists only for the duration of a single autocomplete query. It is created, posted to the event bus, processed by an asynchronous handler, and its results are consumed by the original caller.
- **Destruction:** The object becomes eligible for garbage collection as soon as the results are consumed and all references to it are dropped. It does not persist between frames or UI states.

## Internal State & Concurrency
- **State:** This class is intentionally mutable. While the initial query parameters (editorClient, dataSet, query) are effectively final after construction, the **results** field is designed to be written to *after* instantiation by an asynchronous event handler.
- **Thread Safety:** This class is **not thread-safe** and is not intended to be. It is a data transfer object for a controlled, single-writer, single-reader asynchronous pattern.

    **WARNING:** The internal state, specifically the results array, is unprotected. It is assumed that only one designated handler will call setResults. The thread that created the event must wait for a completion signal from the event system before safely reading the results via getResults. Unsynchronized access will lead to race conditions and unpredictable behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | String | O(1) | Returns the partial text input from the user. |
| getDataSet() | String | O(1) | Returns the identifier for the category of data to be searched (e.g., "block_ids"). |
| getEditorClient() | EditorClient | O(1) | Provides the client context that initiated the request. |
| getResults() | String[] | O(1) | Retrieves the autocomplete suggestions. **Returns null** if the event has not yet been processed. |
| setResults(String[]) | void | O(1) | Populates the event with autocomplete suggestions. Intended for use by the event handler only. |

## Integration Patterns

### Standard Usage
The event is created and dispatched through the event bus. The calling code then awaits the completion of the asynchronous handler to safely access the populated results.

```java
// 1. UI component creates and dispatches the event
AssetEditorFetchAutoCompleteDataEvent event = new AssetEditorFetchAutoCompleteDataEvent(client, "items", "dia");
CompletableFuture<Void> future = eventBus.postAsync(event);

// 2. The UI thread continues rendering while the event is handled in the background.
//    When the result is needed, the future is used.
future.thenAcceptAsync(v -> {
    String[] suggestions = event.getResults();
    // 3. Update the UI with the suggestions
    updateAutoCompleteDropdown(suggestions);
}, mainThreadExecutor);
```

### Anti-Patterns (Do NOT do this)
- **Synchronous Access:** Do not post the event and immediately attempt to read the results. The results field will be null, as the handler has not had time to execute.
    ```java
    // BAD: Race condition, getResults() will almost certainly return null
    eventBus.postAsync(event);
    String[] results = event.getResults(); // This is incorrect
    ```
- **Reusing Instances:** Do not hold onto an event instance and attempt to reuse it for a subsequent query. Each user action must generate a new event object to ensure a clean state.
- **Direct Mutation:** Do not call setResults from any class other than a designated, registered event handler for this event type. Doing so violates the asynchronous request/response pattern.

## Data Pipeline
The event acts as a data carrier in a simple, asynchronous request-response pipeline.

> Flow:
> UI Text Input -> **AssetEditorFetchAutoCompleteDataEvent** created -> Event Bus -> Async Data Service -> Game Asset Registry -> Service calls **setResults()** -> Event Bus signals completion -> UI Thread reads results and updates view

