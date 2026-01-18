---
description: Architectural reference for DropdownEntryInfo
---

# DropdownEntryInfo

**Package:** com.hypixel.hytale.server.core.ui
**Type:** Data Structure / Transient

## Definition
```java
// Signature
public class DropdownEntryInfo {
```

## Architecture & Concepts
The DropdownEntryInfo class is a fundamental data structure representing a single selectable option within a server-driven UI dropdown menu. It is not an active component but rather a passive data container, acting as a data transfer object (DTO) between the server's logic and the client's UI rendering engine.

Its primary architectural role is to serve as a well-defined, serializable contract. The class encapsulates two key pieces of information:
1.  A **label**, represented by a LocalizableString, which is the user-facing text. This design choice is critical for internationalization, allowing the client to render the text in the appropriate language.
2.  A **value**, represented by a standard String, which is the internal, non-localized identifier used by the server to process the user's selection.

The static final field named CODEC is the most significant feature of this class. It exposes a BuilderCodec, which defines the binary or text-based representation of a DropdownEntryInfo instance. This allows the Hytale engine to seamlessly serialize this object for network transmission or persist it in configuration files, ensuring data consistency between the server and client.

## Lifecycle & Ownership
-   **Creation:** An instance of DropdownEntryInfo is created under two distinct circumstances:
    1.  **Programmatically:** Server-side UI generation logic directly instantiates the class using its public constructor to build a list of options for a dropdown component.
    2.  **Deserialization:** The Hytale codec system instantiates the class using the private no-argument constructor when decoding data from a network packet or a configuration file. The CODEC's field definitions are then used to populate the instance.

-   **Scope:** The object is short-lived and transient. Its lifetime is typically bound to the lifetime of the parent UI component definition that contains it. It does not persist beyond a single UI transaction or server session.

-   **Destruction:** The object is managed by the Java garbage collector. There are no native resources or explicit cleanup methods. It is eligible for collection as soon as it is no longer referenced by any active UI model.

## Internal State & Concurrency
-   **State:** The internal state, consisting of the label and value, is mutable. The fields are private, but they are directly modified by the codec during the deserialization process. While not enforced by the language (e.g., with final fields), instances should be treated as immutable after their initial creation.

-   **Thread Safety:** This class is **not thread-safe**. It is a plain data object with no internal locking or synchronization mechanisms. It is designed to be created, populated, and read within a single thread, typically the main server thread.

    **WARNING:** Accessing or modifying a DropdownEntryInfo instance from multiple threads will result in unpredictable behavior and data corruption. Do not share instances across threads without external synchronization.

## API Surface
The public contract is minimal, focusing on creation and serialization. Direct access to internal fields is intentionally restricted.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DropdownEntryInfo(label, value) | constructor | O(1) | Constructs a new dropdown entry. |
| CODEC | BuilderCodec | O(1) | Static field providing the serializer and deserializer for this class. |

## Integration Patterns

### Standard Usage
The class is used to populate a collection of options that will be part of a larger UI component.

```java
// How a developer should normally use this
List<DropdownEntryInfo> options = new ArrayList<>();

options.add(new DropdownEntryInfo(
    LocalizableString.of("ui.options.difficulty.easy"),
    "easy"
));

options.add(new DropdownEntryInfo(
    LocalizableString.of("ui.options.difficulty.hard"),
    "hard"
));

// This list of options would then be passed to a parent UI component builder.
```

### Anti-Patterns (Do NOT do this)
-   **Post-Creation Modification:** Do not attempt to modify the state of a DropdownEntryInfo object after it has been created. It should be treated as an immutable value object.
-   **Misuse as a General-Purpose Pair:** This class is specifically for UI dropdowns. Do not use it as a generic key-value pair container in other parts of the system; use a more appropriate data structure.

## Data Pipeline
DropdownEntryInfo serves as a payload in the server-to-client UI data flow. Its journey is governed by the codec system.

> Flow:
> Server UI Logic -> **new DropdownEntryInfo()** -> Serialization via CODEC -> Network Packet -> Client Network Layer -> Deserialization via CODEC -> **DropdownEntryInfo instance** -> Client UI Renderer

