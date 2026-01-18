---
description: Architectural reference for LocalizableString
---

# LocalizableString

**Package:** com.hypixel.hytale.server.core.ui
**Type:** Value Object / DTO

## Definition
```java
// Signature
public class LocalizableString {
```

## Architecture & Concepts
The LocalizableString class is a foundational data structure within the UI and internationalization (i18n) framework. It represents a piece of text that can exist in one of two states: a raw, literal string or a structured, localizable message identifier with parameters. This dual-state design provides critical flexibility for developers and content creators.

Its primary architectural role is to serve as a standardized container for any user-facing text. By abstracting the concept of a string, the system can transparently handle text from various sources—be it hardcoded in a configuration file or dynamically looked up from a language-specific translation table.

The class is deeply integrated with the Hytale Codec system. It defines a custom polymorphic codec, LocalizableStringCodec, which can intelligently serialize and deserialize the object to and from BSON. It can encode to a simple BsonString for literal values, optimizing for space and performance, or to a full BSON object for localizable messages. This makes LocalizableString a ubiquitous data type in network packets, asset files, and server configurations.

## Lifecycle & Ownership
-   **Creation:** Instances are never created directly via a constructor. They are instantiated exclusively through one of two mechanisms:
    1.  **Static Factories:** The public static methods fromString and fromMessageId are the primary entry points for programmatic creation.
    2.  **Deserialization:** The Hytale Codec engine automatically instantiates LocalizableString objects when decoding BSON data from network streams or configuration files, using the registered LocalizableStringCodec.

-   **Scope:** A LocalizableString is a transient value object. Its lifetime is bound to the lifetime of the object that contains it, such as a UI component definition, an entity's nameplate data, or a network message. It holds no global state and does not persist beyond its containing scope.

-   **Destruction:** Instances are managed by the Java Garbage Collector. There are no native resources or explicit cleanup steps required.

## Internal State & Concurrency
-   **State:** Instances are **effectively immutable**. While the internal fields are not declared final, there are no public mutators (setters). State is established once at creation time by the static factory methods or the codec. An instance will either have its stringValue field populated or its messageId and messageParams fields populated, but never both.

-   **Thread Safety:** The class is inherently **thread-safe**. Due to its immutable nature, an instance of LocalizableString can be safely shared and read across multiple threads without locks or other synchronization primitives.

## API Surface
The public contract consists entirely of static factory methods for object creation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fromString(String str) | LocalizableString | O(1) | Creates an instance representing a raw, literal string. |
| fromMessageId(String id) | LocalizableString | O(1) | Creates an instance representing a localizable message key. |
| fromMessageId(String id, Map params) | LocalizableString | O(1) | Creates an instance representing a localizable message key with substitution parameters. |

## Integration Patterns

### Standard Usage
A LocalizableString should be used whenever defining user-facing text that may require translation. It is typically passed to UI components or other systems that are aware of the localization pipeline.

```java
// Example: Setting the title of a UI dialog
// This message key will be resolved by the localization engine.
LocalizableString dialogTitle = LocalizableString.fromMessageId(
    "ui.dialog.confirm_exit.title"
);

// This message includes parameters for substitution.
Map<String, String> params = new HashMap<>();
params.put("worldName", "My Awesome World");
LocalizableString dialogBody = LocalizableString.fromMessageId(
    "ui.dialog.confirm_exit.body",
    params
);

// The UI system would then use these objects to render the final text.
uiDialog.setTitle(dialogTitle);
uiDialog.setBody(dialogBody);
```

### Anti-Patterns (Do NOT do this)
-   **Manual Resolution:** Do not attempt to manually parse the messageId or resolve the string yourself. Pass the LocalizableString object directly to systems designed to handle it, such as the UI rendering engine or a dedicated LocalizationService. The internal state is an implementation detail.

-   **Non-UI Text:** Avoid using this class for internal system strings, log messages, or identifiers that are not meant for player display. It introduces unnecessary overhead compared to a standard Java String.

-   **Complex Parameter Types:** The messageParams map is designed for String-to-String substitution. Do not store complex objects as parameters; serialize them to strings before creating the LocalizableString.

## Data Pipeline
The primary data flow for a LocalizableString involves its deserialization from a data source, resolution by a localization service, and final rendering in the UI.

> Flow:
> BSON Data (Network Packet / Asset File) -> Hytale Codec Engine -> **LocalizableString Instance** -> Localization Service -> Final Rendered String -> UI Engine

