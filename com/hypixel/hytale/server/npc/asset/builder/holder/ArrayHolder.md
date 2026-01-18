---
description: Architectural reference for ArrayHolder
---

# ArrayHolder

**Package:** com.hypixel.hytale.server.npc.asset.builder.holder
**Type:** Component / Abstract Base Class

## Definition
```java
// Signature
public abstract class ArrayHolder extends ValueHolder {
```

## Architecture & Concepts
The **ArrayHolder** is an abstract base class that serves as a foundational component within the server-side NPC asset building system. Its primary responsibility is to manage the structural validation and parsing logic for array-based properties defined in JSON configuration files.

This class extends **ValueHolder**, specializing its behavior to handle collections. It does not parse the contents of the array itself; rather, it establishes and enforces constraints on the array's *size*. The actual parsing of elements (e.g., numbers, strings) is delegated to concrete subclasses.

Architecturally, **ArrayHolder** embodies the Template Method design pattern. The various `readJSON` methods provide a skeleton algorithm:
1.  Configure the required length constraints (**minLength**, **maxLength**).
2.  Delegate the core JSON parsing and expression creation to the superclass or a subclass implementation.
3.  Provide a `validateLength` hook for subclasses to call back to, ensuring the parsed array conforms to the configured constraints.

This design effectively decouples array size validation from the logic of parsing the array's contents, promoting reuse and maintainability.

## Lifecycle & Ownership
-   **Creation:** As an abstract class, **ArrayHolder** is never instantiated directly. Concrete subclasses, such as **NumberArrayHolder** or **StringArrayHolder**, are created by a higher-level builder class during the deserialization of an NPC asset definition.
-   **Scope:** An instance of an **ArrayHolder** subclass is ephemeral. Its lifecycle is strictly bound to the parsing operation of a single JSON field within a single asset build process. It does not persist after the asset has been constructed.
-   **Destruction:** The object is eligible for garbage collection as soon as the asset builder completes its operation and all references to it are dropped. There are no explicit cleanup or disposal methods.

## Internal State & Concurrency
-   **State:** The class maintains mutable state in the form of two protected integer fields: **minLength** and **maxLength**. This state is configured once via a `setLength` call, which is typically invoked at the beginning of a `readJSON` operation. Once set, this state is considered final for the lifetime of the parsing operation.

-   **Thread Safety:** **ArrayHolder** is **not thread-safe**. It is designed for use within a single-threaded, sequential asset-building pipeline. Concurrent calls to `readJSON` or `setLength` on the same instance would result in a race condition, leading to unpredictable and incorrect validation behavior. The entire NPC asset builder system is presumed to operate on a single thread.

## API Surface
The primary API consists of protected methods intended for use by concrete subclasses and the builder system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readJSON(...) | protected void | O(1) | Overloaded methods to initiate parsing. Sets length constraints and delegates to the superclass. |
| validateLength(int length) | protected void | O(1) | Core validation hook. Throws **IllegalStateException** if the provided length is outside the configured min/max bounds. |
| setLength(int min, int max) | protected void | O(1) | Configures the valid length range for the array. Throws **IllegalArgumentException** for invalid ranges. |

## Integration Patterns

### Standard Usage
A concrete implementation of **ArrayHolder** is used by a parent builder to parse a specific JSON array field. The builder provides the JSON data, the required length constraints, and a default value if the field is optional.

```java
// Hypothetical usage within a parent builder class
// A concrete subclass like NumberArrayHolder would be used in practice.

// 1. Obtain the JSON array element from the parent object
JsonObject parentJson = ...;
JsonElement arrayElement = parentJson.get("positions");

// 2. Instantiate the specific holder
// NOTE: This would be a concrete class, not the abstract ArrayHolder
NumberArrayHolder positionHolder = new NumberArrayHolder();

// 3. Invoke readJSON with constraints
// This requires the array to have exactly 3 elements.
positionHolder.readJSON(arrayElement, 3, 3, "positions", builderParameters);
```

### Anti-Patterns (Do NOT do this)
-   **Reusing Instances:** An **ArrayHolder** instance is stateful and configured for a single JSON property. Do not reuse the same instance to parse a different property, as its internal length constraints will be incorrect for the new context.
-   **Bypassing `readJSON`:** Manually calling `setLength` and then attempting to orchestrate the parsing logic from the outside violates the intended flow. The `readJSON` methods are the correct entry points.
-   **Concurrent Access:** Do not share an **ArrayHolder** instance across multiple threads. The asset-building process must be sequential.

## Data Pipeline
**ArrayHolder** sits in the middle of the JSON-to-Asset pipeline. It acts as a specialized validator and delegator for array properties.

> Flow:
> Raw JSON File -> GSON Parser -> Parent Builder -> **Concrete ArrayHolder Subclass** -> `validateLength` check -> **BuilderExpression** -> Compiled NPC Asset

