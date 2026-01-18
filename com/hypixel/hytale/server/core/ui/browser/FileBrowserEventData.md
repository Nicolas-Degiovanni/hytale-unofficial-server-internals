---
description: Architectural reference for FileBrowserEventData
---

# FileBrowserEventData

**Package:** com.hypixel.hytale.server.core.ui.browser
**Type:** Transient Data Object

## Definition
```java
// Signature
public class FileBrowserEventData {
```

## Architecture & Concepts
The FileBrowserEventData class is a specialized Data Transfer Object (DTO) designed to represent the payload of UI events originating from an in-game file browser. It serves as a structured data contract between a front-end view, likely a web-based UI, and the server-side logic that handles file system interactions.

Its primary architectural role is to be a deserialization target for the Hytale **Codec** system. The static CODEC field defines a rigid schema for converting raw, untyped data (e.g., from a JSON-like structure) into a strongly-typed Java object. This pattern ensures that all file browser events conform to a predictable structure, preventing data corruption and simplifying event handling logic. This class does not contain any business logic; its sole responsibility is to carry state.

## Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the Hytale serialization framework. The public static field **CODEC** is used by an event dispatcher or network layer to decode an incoming data payload into a new FileBrowserEventData object. Manual instantiation via the static factory method *file(String)* is supported for specific, programmatic event generation.

- **Scope:** The object's lifetime is extremely short. It is scoped to the processing of a single UI event. It is created, passed to an event handler, read, and then becomes eligible for garbage collection.

- **Destruction:** The object holds no native resources and requires no explicit cleanup. It is managed entirely by the Java garbage collector once all references to it are dropped at the end of the event handling cycle.

## Internal State & Concurrency
- **State:** The internal state is mutable. The fields are populated by the CODEC during the deserialization process. The object is not designed to be modified after its initial creation.

- **Thread Safety:** This class is **not thread-safe**. It is a simple data container with no internal synchronization mechanisms. Instances should be confined to the thread that is processing the UI event. Sharing an instance across multiple threads without external synchronization will lead to unpredictable behavior and race conditions.

## API Surface
The public API is minimal, focusing on data access and the critical serialization contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | BuilderCodec | O(N) | **Critical.** The public contract for serializing and deserializing instances of this class. N is the number of fields. |
| getFile() | String | O(1) | Returns the file path associated with the event, if any. |
| getRoot() | String | O(1) | Returns the root directory for a browse or search operation. |
| getSearchQuery() | String | O(1) | Returns the user-provided search term. |
| getSearchResult() | String | O(1) | Returns a specific search result selected by the user. |
| isBrowseRequested() | boolean | O(1) | Returns true if the event represents a request to browse a directory. |
| file(String) | FileBrowserEventData | O(1) | A static factory for creating a simple event object representing a file selection. |

## Integration Patterns

### Standard Usage
The intended use is to receive this object from a higher-level system, such as an event bus, after it has been deserialized. The handler then inspects the object's state to determine the correct action.

```java
// In an event handler method
public void onFileBrowserAction(FileBrowserEventData eventData) {
    if (eventData.isBrowseRequested()) {
        String rootPath = eventData.getRoot();
        // Logic to list files in the rootPath
    } else if (eventData.getFile() != null) {
        String selectedFile = eventData.getFile();
        // Logic to process the selected file
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new FileBrowserEventData()`. This bypasses the intended creation pathways (CODEC deserialization or the `file` factory method) and results in an empty, useless object.

- **State Mutation:** Do not modify the state of a FileBrowserEventData object after it has been created and passed to a handler. It should be treated as an immutable record for the duration of its lifecycle.

- **Cross-Thread Sharing:** Never pass an instance of this object to another thread for processing. If multi-threaded work is required, extract the primitive data (e.g., the file path string) and pass that instead.

## Data Pipeline
FileBrowserEventData acts as a translation layer, converting a raw data stream into a structured, usable object within the server's domain.

> Flow:
> UI User Action -> Serialized Event Payload (e.g., JSON) -> **FileBrowserEventData.CODEC Deserialization** -> **FileBrowserEventData Instance** -> Server Event Bus -> Event Handler Logic

