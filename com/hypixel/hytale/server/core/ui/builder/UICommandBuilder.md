---
description: Architectural reference for UICommandBuilder
---

# UICommandBuilder

**Package:** com.hypixel.hytale.server.core.ui.builder
**Type:** Transient

## Definition
```java
// Signature
public class UICommandBuilder {
```

## Architecture & Concepts

The UICommandBuilder is a server-side fluent API responsible for constructing sequences of commands that dynamically manipulate a client's user interface. It acts as a high-level abstraction over the raw `CustomUICommand` packet structure, enabling developers to describe UI changes declaratively.

Its primary architectural function is to serve as a serialization bridge. It translates server-side Java objects and primitive types into a BSON (Binary JSON) representation. This BSON data is then embedded within individual UI commands. The client-side UI engine receives these commands and uses the BSON payload to update the state and appearance of UI components.

This system allows for highly dynamic and responsive interfaces without requiring a full UI document reload for minor changes. The builder uses a string-based selector model, conceptually similar to CSS selectors, to target specific elements within the client's UI document for operations like appending content, removing elements, or setting properties.

A core component of this architecture is the static `CODEC_MAP`, which registers BSON serializers (Codecs) for specific Hytale data types like `ItemStack`, `Area`, and `LocalizableString`. This registry allows the `setObject` method to transparently handle the serialization of complex game objects into a format the client can understand.

## Lifecycle & Ownership

-   **Creation:** A new UICommandBuilder is instantiated on-demand by server-side logic. It is not a managed service or singleton. It should be created whenever a batch of UI modifications is required, typically within a single method's scope.
-   **Scope:** The lifecycle of a UICommandBuilder instance is intentionally short-lived and transient. It exists only for the duration of building one specific set of UI commands.
-   **Destruction:** The object is intended to be discarded and made eligible for garbage collection immediately after the `getCommands` method is called. The returned array of commands is then passed to the network layer for transmission.

## Internal State & Concurrency

-   **State:** The internal state consists of a mutable `List` of `CustomUICommand` objects. Each call to a builder method such as `append` or `set` adds a new command to this internal list, progressively building the final command sequence.
-   **Thread Safety:** **This class is not thread-safe.** It is designed for synchronous, single-threaded use. Sharing a single UICommandBuilder instance across multiple threads will lead to race conditions and a corrupted command list, causing unpredictable and erroneous UI behavior on the client. Each thread or logical task must create and operate on its own distinct instance.

## API Surface

The API is designed as a fluent interface, where most methods return the builder instance to allow for chaining.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| append(selector, documentPath) | UICommandBuilder | O(1) | Appends a new UI element, defined in a separate document, as a child of the selected element. |
| appendInline(selector, document) | UICommandBuilder | O(1) | Appends a new UI element, defined by an inline string, as a child of the selected element. |
| set(selector, value) | UICommandBuilder | O(k) | Sets a property on the selected element. Overloaded for various primitives and Hytale types. Complexity depends on BSON encoding. |
| setObject(selector, data) | UICommandBuilder | O(k) | Sets a property using a registered BSON codec for a complex object. Throws `IllegalArgumentException` if no codec is found. |
| getCommands() | CustomUICommand[] | O(N) | Finalizes the build process and returns the complete, ordered array of UI commands. N is the number of commands built. |

## Integration Patterns

### Standard Usage

The builder is used to construct a command payload which is then typically sent to a client via a network packet.

```java
// Example: Updating a player's quest log UI
UICommandBuilder builder = new UICommandBuilder();

// Atomically clear the old list and add new items
builder.clear("#quest-list")
       .appendInline("#quest-list", "<Quest title='Find the Lost Sword'/>")
       .set("#quest-list .title.text", "A Blade of Legend")
       .set("#quest-list .title.enabled", true);

// Retrieve the command array for network transmission
CustomUICommand[] commands = builder.getCommands();

// The command array is then wrapped in a packet and sent to the player
// player.getConnection().send(new PacketCustomUI(commands));
```

### Anti-Patterns (Do NOT do this)

-   **Instance Re-use:** Do not reuse a UICommandBuilder instance after calling `getCommands`. The internal list is not cleared, and subsequent calls will append to the existing list, leading to unintended and duplicate UI operations. Always create a new builder for each new transaction.
-   **Concurrent Modification:** Do not share a builder instance across multiple threads. This will corrupt the internal command list. Each thread must have its own builder.
-   **Large Payloads in `set`:** While convenient, serializing extremely large or complex objects via `setObject` can create oversized network packets. For large data transfers, consider alternative mechanisms or break the data into smaller, targeted updates.

## Data Pipeline

The UICommandBuilder is an early step in the server-to-client UI update pipeline. It transforms high-level server state into a low-level, serializable command format.

> Flow:
> Server-Side Game Logic -> **UICommandBuilder** -> BSON Serialization -> CustomUICommand Array -> Network Packet -> Client UI Engine -> Rendered UI Update

