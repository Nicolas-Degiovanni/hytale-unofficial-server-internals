---
description: Architectural reference for ComponentInfo
---

# ComponentInfo

**Package:** com.hypixel.hytale.server.npc.util
**Type:** Transient

## Definition
```java
// Signature
public class ComponentInfo {
```

## Architecture & Concepts
ComponentInfo is a transient Data Transfer Object (DTO) designed to capture and format the state of a game entity's component for diagnostic and debugging purposes. Its primary function is to act as a temporary data structure for a system that traverses an entity's component hierarchy.

The class aggregates details such as the component's name, its index within a list, its nesting depth for hierarchical display, and a list of its fields. The core value of this class is its `toString` method, which serializes this captured state into a human-readable, indented format suitable for logging or display in a debug console. It is a fundamental utility for providing introspection into the server-side NPC component system.

## Lifecycle & Ownership
- **Creation:** Instances are created directly via the public constructor: `new ComponentInfo(...)`. This is typically done by a higher-level diagnostic or serialization service that is actively inspecting an entity's live component data.
- **Scope:** Ephemeral and short-lived. A ComponentInfo object exists only for the duration of the data-gathering and formatting operation. It is not intended to be stored or cached.
- **Destruction:** The object has no explicit cleanup logic. It becomes eligible for garbage collection as soon as the reference to it is dropped, which is usually immediately after its string representation has been generated and consumed.

## Internal State & Concurrency
- **State:** The object's state is **Mutable**. While the `name`, `index`, and `nestingDepth` fields are final and set at construction, the internal `fields` list is populated post-construction via the `addField` method. The object is designed to be built up over a series of calls before its final state is read.
- **Thread Safety:** This class is **not thread-safe**. The internal `fields` list is an `ObjectArrayList`, which is not synchronized. Concurrent calls to `addField` or iterating the list from another thread while it is being modified will lead to undefined behavior, including `ConcurrentModificationException`. Synchronization must be handled externally by the calling context if an instance is ever shared across threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ComponentInfo(name, index, nestingDepth) | constructor | O(1) | Creates a new instance to represent a component. |
| addField(field) | void | O(1) amortized | Appends a string representation of a component field to the internal list. |
| toString() | String | O(N) | Generates a formatted, indented string representing the component and its fields. N is the number of fields. |
| getName() | String | O(1) | Returns the component's name. |
| getIndex() | int | O(1) | Returns the component's index, or -1 if not applicable. |
| getFields() | List<String> | O(1) | Returns a direct reference to the internal list of fields. **Warning:** Modifying this list externally is an anti-pattern. |

## Integration Patterns

### Standard Usage
The intended pattern is to create, populate, and immediately consume the object within a single method scope. It acts as a builder for a formatted debug string.

```java
// A diagnostic utility would create and populate this object
ComponentInfo info = new ComponentInfo("TransformComponent", 0, 1);
info.addField("position: {x: 10, y: 50, z: 20}");
info.addField("rotation: {yaw: 90.0}");

// The result is then consumed, typically for logging
System.out.println(info.toString());
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not hold onto a ComponentInfo instance and attempt to clear its fields for re-use. The object is lightweight and should be discarded after use. Create a new instance for each component you inspect.
- **Concurrent Modification:** Never share a ComponentInfo instance across multiple threads without external locking. The internal list is not thread-safe.
- **External List Mutation:** Do not modify the list returned by `getFields`. This method is provided for read-only inspection, and mutating the list directly can corrupt the object's state.

## Data Pipeline
ComponentInfo serves as a formatting stage in a diagnostic data pipeline. It does not process logic but rather shapes data for presentation.

> Flow:
> Live Entity Component Data -> Diagnostic Traversal System -> **ComponentInfo (Builder)** -> Formatted String -> Log File / Debug Console

