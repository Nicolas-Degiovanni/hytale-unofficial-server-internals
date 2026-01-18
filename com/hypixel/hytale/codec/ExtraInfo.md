---
description: Architectural reference for ExtraInfo
---

# ExtraInfo

**Package:** com.hypixel.hytale.codec
**Type:** Transient Context Object

## Definition
```java
// Signature
public class ExtraInfo {
```

## Architecture & Concepts
The **ExtraInfo** class is a foundational component of the Hytale codec framework, serving as a stateful context object during data serialization and deserialization operations. Its primary role is to track the traversal path, manage versioning, and aggregate validation results while a codec processes a data structure, typically from a JSON source.

Architecturally, it employs the **Thread Confinement** pattern via a **ThreadLocal** static field. This design ensures that each thread performing a codec operation receives its own isolated instance of **ExtraInfo**. This is a critical performance and safety feature, eliminating both the overhead of passing the context object through every method in the call stack and the risk of race conditions in a multi-threaded environment, such as a server loading multiple assets concurrently.

The class maintains an internal stack of keys (both string and integer) to construct a fully-qualified path to the current data element being processed. This path is indispensable for generating precise and actionable error messages, pointing developers to the exact location of a data validation failure.

## Lifecycle & Ownership
- **Creation:** An **ExtraInfo** instance is implicitly created and managed by its static **ThreadLocal** field. The first time a thread calls **ExtraInfo.THREAD_LOCAL.get()**, a new **ExtraInfo** object is instantiated via its default constructor and associated with that thread.

- **Scope:** The object's lifecycle is bound to a single, continuous codec operation within a single thread. It is effectively a transient, per-operation state container.

- **Destruction:** The object is eligible for garbage collection when its owning thread terminates. There are no explicit cleanup or disposal methods, as its state is not intended to persist beyond the scope of one decoding task.

## Internal State & Concurrency
- **State:** The internal state of **ExtraInfo** is highly **mutable**. It is fundamentally a state machine whose properties—such as the key stack and list of unknown keys—are constantly modified as the codec navigates a data graph. For performance, it utilizes primitive-backed parallel arrays that grow dynamically, avoiding the overhead of wrapper objects.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. The architecture enforces safety by providing each thread its own instance through **ThreadLocal**. Any attempt to manually pass an **ExtraInfo** instance to another thread will result in undefined behavior and state corruption.

## API Surface
The public API is designed for state manipulation by a controlling codec during a recursive-descent parse.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| THREAD_LOCAL.get() | ExtraInfo | O(1) | Retrieves the instance of **ExtraInfo** for the current thread. |
| pushKey(key) | void | O(1) amortized | Pushes a string key onto the internal path stack. May trigger array resizing. |
| pushIntKey(key) | void | O(1) amortized | Pushes an integer key onto the internal path stack, used for array indices. |
| popKey() | void | O(1) | Pops the most recent key from the internal path stack. |
| peekKey() | String | O(N) | Constructs and returns the full, dot-separated path based on the current stack. |
| addUnknownKey(key) | void | O(1) amortized | Registers a key that was found in the source data but was not expected by the codec. |
| getValidationResults() | ValidationResults | O(1) | Returns the handler responsible for processing validation failures. |

## Integration Patterns

### Standard Usage
A codec should always retrieve the context object from the **ThreadLocal** at the beginning of a decoding operation. It is responsible for managing the key stack as it traverses the data.

```java
// Standard pattern for a codec decoding an object
ExtraInfo extraInfo = ExtraInfo.THREAD_LOCAL.get();

// Before decoding a field "playerName"
extraInfo.pushKey("playerName");
String name = reader.readString();
// After decoding the field
extraInfo.popKey();
```

### Anti-Patterns (Do NOT do this)
- **Cross-Thread Sharing:** Never get an instance on one thread and pass it to another. This breaks the thread-confinement model and will lead to data corruption.

- **Improper Stack Management:** Failing to call **popKey** after processing a field will corrupt the path for all subsequent sibling fields, leading to misleading error reports. Every **pushKey** must be paired with a corresponding **popKey** within the same scope.

- **Direct Instantiation:** Avoid `new ExtraInfo()` unless you are managing its lifecycle manually in a single-threaded context. The framework relies on the **ThreadLocal** instance for integration with other systems.

## Data Pipeline
**ExtraInfo** acts as a sidecar for data flowing through a codec. It does not transform the data itself but rather observes the process to provide context, validation, and error reporting.

> Flow:
> Raw Data Stream (e.g., JSON file) -> RawJsonReader -> Codec.decode() -> **ExtraInfo** (tracks path and errors) -> Hydrated Java Object

