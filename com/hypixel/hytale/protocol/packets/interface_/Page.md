---
description: Architectural reference for Page
---

# Page

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Utility

## Definition
```java
// Signature
public enum Page {
```

## Architecture & Concepts
The Page enum serves as a type-safe, integer-backed contract for identifying core user interface screens or "pages" within the game client. Its primary function is to translate low-level integer identifiers, received over the network protocol, into high-level, self-documenting constants.

This component is critical for protocol stability and maintainability. By defining a fixed set of possible UI states, it decouples the network layer from the UI implementation. The server can command the client to open a specific page, such as Inventory (2), without any knowledge of the client's internal UI management system. This prevents the use of "magic numbers" in the networking code, reducing ambiguity and the potential for deserialization errors.

The inclusion of a static factory method, fromValue, centralizes the logic for validating and converting incoming network data into a valid Page instance.

### Lifecycle & Ownership
- **Creation:** All enum constants (None, Bench, Inventory, etc.) are instantiated by the Java Virtual Machine during class loading. This process is automatic and occurs exactly once when the Page class is first referenced.
- **Scope:** Application-wide. Once loaded, the enum constants persist for the entire lifetime of the application. They are effectively static, global singletons.
- **Destruction:** The enum and its constants are garbage collected only when the JVM shuts down. There is no mechanism for manual destruction.

## Internal State & Concurrency
- **State:** The Page enum is **immutable**. Each constant holds a single, final integer field named value. The static VALUES array, used for efficient lookups, is also final and populated only once at class initialization. The state of a Page constant can never change after creation.
- **Thread Safety:** This class is inherently thread-safe. Due to its immutability and the JVM's guarantees regarding enum initialization, instances of Page can be safely accessed, passed, and read from any thread without requiring external synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer value used for network serialization. |
| fromValue(int value) | Page | O(1) | **[Static]** Deserializes an integer into a Page constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
The canonical use case for Page is during packet deserialization. An integer is read from the network buffer and converted into a Page object, which is then passed to the game's UI logic.

```java
// Example from a hypothetical packet handler
int pageId = buffer.readVarInt();
try {
    Page targetPage = Page.fromValue(pageId);
    
    // Pass the type-safe enum to the UI system
    uiManager.openPage(targetPage);
} catch (ProtocolException e) {
    // Disconnect the client for sending malformed data
    networkSession.disconnect("Invalid page ID: " + pageId);
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Exceptions:** The fromValue method is a critical validation boundary. Failure to catch the ProtocolException it throws can lead to unhandled exceptions in the network processing thread, potentially crashing the client or server. Callers **MUST** handle this exception.
- **Using ordinal() for Serialization:** Under no circumstances should the built-in enum ordinal() method be used for serialization or deserialization. The protocol relies on the explicit integer `value` field. Relying on ordinal() would create a brittle system where simply reordering the enum declarations in this file would break network compatibility.
- **Hard-coding Integer Values:** Avoid using raw integers (e.g., `uiManager.openPage(2)`) in game logic. Always reference the enum constants (e.g., `uiManager.openPage(Page.Inventory)`) to maintain type safety and readability.

## Data Pipeline
The Page enum acts as a translation and validation step in the data flow from the network to the user interface.

> Flow:
> Network Byte Stream -> Protocol Deserializer -> **Page.fromValue()** -> Page Enum Instance -> UI Management System -> Render Target UI Screen

