---
description: Architectural reference for Value
---

# Value

**Package:** com.hypixel.hytale.server.core.ui
**Type:** Utility / Value Object

## Definition
```java
// Signature
public class Value<T> {
```

## Architecture & Concepts
The Value class is a generic container that implements the **Value or Reference** pattern. Its fundamental purpose is to represent a piece of data that can exist in one of two mutually exclusive states:

1.  **A Concrete Value:** The object directly holds an instance of type T. This is used when the data is known at creation time.
2.  **A Symbolic Reference:** The object holds a path and a name that point to a value stored elsewhere, typically within a larger data document or configuration system. This is used for late-binding of data, allowing UI or game logic to be defined generically without being coupled to the immediate data source.

This pattern is critical for decoupling systems. For instance, a UI template can be defined with references to game state (e.g., `Value.ref("player.stats", "health")`). A separate data-binding system is then responsible for resolving this reference at runtime and populating the UI with the actual health value. This makes the UI definition reusable and independent of the underlying data model's lifecycle.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively through the static factory methods `Value.of(T value)` or `Value.ref(String document, String value)`. The constructors are private to enforce the integrity of the two distinct states and prevent the creation of invalid or ambiguous objects.
-   **Scope:** Value objects are transient and short-lived. They are designed to be passed by value between systems and do not own any resources. Their lifetime is typically bound to the scope of the method that creates them or the object that holds them (e.g., a UI component model).
-   **Destruction:** As simple data holders with no external resource handles, instances are managed entirely by the Java Garbage Collector. No explicit cleanup is necessary.

## Internal State & Concurrency
-   **State:** The Value class is **immutable**. Its internal state (either the direct value or the reference path) is set once during construction and cannot be modified thereafter. This design guarantees that a Value object's state is consistent and predictable throughout its lifetime.
-   **Thread Safety:** Due to its immutability, the Value class is inherently **thread-safe**. Instances can be safely shared and read by multiple threads without any external synchronization or locking mechanisms.

## API Surface
The public API is minimal, focusing on object creation and state inspection.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ref(document, value) | static `Value<T>` | O(1) | Factory method. Creates a new Value instance that acts as a symbolic reference. |
| of(value) | static `Value<T>` | O(1) | Factory method. Creates a new Value instance that holds a concrete value. |
| getValue() | T | O(1) | Returns the concrete value. Returns null if this is a reference-type instance. |
| getDocumentPath() | String | O(1) | Returns the document path for a reference. Returns null if this is a value-type instance. |
| getValueName() | String | O(1) | Returns the value name for a reference. Returns null if this is a value-type instance. |

## Integration Patterns

### Standard Usage
The consuming code must inspect the state of a Value object to determine whether it holds a direct value or a reference, and then act accordingly.

```java
// Example of creating a direct value for a UI label
Value<String> title = Value.of("Main Menu");

// Example of creating a reference to a dynamic value
Value<Integer> health = Value.ref("player.stats", "currentHealth");

// A hypothetical consumer system must handle both cases
public void updateUI(Value<?> data) {
    if (data.getDocumentPath() != null) {
        // This is a reference, resolve it
        Object resolvedValue = dataResolver.fetch(data.getDocumentPath(), data.getValueName());
        render(resolvedValue);
    } else {
        // This is a direct value, use it immediately
        render(data.getValue());
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** The constructors are private, so `new Value()` will result in a compile-time error. This is by design to prevent the creation of objects in an invalid state.
-   **Unchecked Access:** Never assume the type of a Value object. Calling `getValue()` on a reference-type instance will return null, potentially leading to NullPointerExceptions downstream. Always check for the presence of a document path first.

    ```java
    // BAD: This can throw a NullPointerException
    Value<Integer> healthRef = Value.ref("player.stats", "currentHealth");
    int currentHealth = healthRef.getValue(); // This returns null!

    // GOOD: Check the state before accessing
    if (healthRef.getDocumentPath() == null) {
        int currentHealth = healthRef.getValue();
        // ...
    }
    ```

## Data Pipeline
The Value class does not process data itself; it is a data structure used to define a data flow, particularly for data binding.

> Flow:
> UI Definition (e.g., JSON) -> Parser -> Creates **Value.ref("path", "name")** -> Data Binding System -> Resolves Reference Against Game State -> Fetches Real Data -> UI Component Update

