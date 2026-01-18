---
description: Architectural reference for IntArg
---

# IntArg

**Package:** com.hypixel.hytale.server.core.asset.type.buildertool.config.args
**Type:** Transient

## Definition
```java
// Signature
public class IntArg extends ToolArg<Integer> {
```

## Architecture & Concepts
The IntArg class is a server-side data model that defines the schema and constraints for an integer-based argument within the Builder Tools system. It is not merely a container for a value; it is a complete blueprint that specifies a default value, a minimum bound, and a maximum bound.

This class serves as a critical bridge between three distinct engine systems:
1.  **Asset Configuration:** Through its static CODEC field, it enables declarative definition of tool arguments in external asset files, which are deserialized at runtime.
2.  **Command Processing:** It provides validation logic via the fromString method to parse and constrain user input received from commands or other interactions.
3.  **Network Synchronization:** It handles the serialization of its definition into a network packet (BuilderToolIntArg) to inform the client UI about the argument's properties, such as rendering a slider with the correct min/max range.

IntArg is a concrete implementation of the generic ToolArg, specializing the behavior for integer types.

## Lifecycle & Ownership
-   **Creation:** Instances are primarily created by the engine's Codec system during server bootstrap or asset reloading. The static CODEC field is used to deserialize an IntArg definition from a Builder Tool asset file. Direct programmatic instantiation is rare and reserved for dynamically generated tools.
-   **Scope:** The lifetime of an IntArg instance is bound to its parent Builder Tool configuration. It is loaded into memory when the tool asset is loaded and persists as an immutable definition for the entire server session.
-   **Destruction:** The object is marked for garbage collection when its parent Builder Tool asset is unloaded, which typically occurs on server shutdown or during a comprehensive asset hot-reload.

## Internal State & Concurrency
-   **State:** The internal state (min, max, and the default value) is effectively **immutable** post-deserialization. While the fields are not declared final, they are set once during the object's construction by the codec and are not exposed for modification thereafter. This design ensures that a tool's definition remains consistent throughout its lifecycle.
-   **Thread Safety:** This class is **thread-safe**. Its immutable nature allows a single instance to be safely read and used by multiple player threads simultaneously without requiring any locks or synchronization. For example, two players using the same tool will reference the same IntArg definition to validate their distinct inputs.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fromString(String str) | Integer | O(1) | Parses a string input into an integer and validates it against the min/max bounds. Throws ToolArgException if parsing fails or the value is out of range. |
| toIntArgPacket() | BuilderToolIntArg | O(1) | Creates a new, specialized network packet containing the default value, min, and max. This is used to synchronize the argument's definition to the client. |
| setupPacket(BuilderToolArg packet) | void | O(1) | Populates a generic BuilderToolArg packet with the specific integer argument data. This is part of a polymorphic networking pattern. |

## Integration Patterns

### Standard Usage
The IntArg object is almost never used directly. Instead, the engine's Builder Tool service retrieves it from a loaded tool definition to validate input or synchronize state with the client.

```java
// PSEUDOCODE: Engine validating a player's input for a tool
BuilderTool tool = toolRegistry.get("my_awesome_tool");
IntArg sizeArg = (IntArg) tool.getArgument("size");

try {
    // Player inputs "50" via a command
    Integer validatedSize = sizeArg.fromString("50");
    tool.applyEffect(validatedSize);
} catch (ToolArgException e) {
    player.sendMessage(e.getTranslatedMessage());
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Avoid `new IntArg()`. Tool arguments should be defined declaratively in asset files. This ensures that configuration is centralized and data-driven, rather than hardcoded.
-   **State Mutation:** Do not use reflection or other means to modify the min or max fields after an instance has been created. This violates the class's design contract as an immutable definition and can lead to inconsistent validation behavior across the server.

## Data Pipeline
The IntArg class functions at several key points in the data pipeline, transforming data from configuration files to network packets.

> **Configuration Flow:**
> Asset File (e.g., JSON) -> Hytale Codec System -> **IntArg Instance** (In-memory representation)

> **User Input Validation Flow:**
> Player Command Input (String) -> `fromString` method on **IntArg** -> Validated Integer or ToolArgException

> **Client Sync Flow:**
> Server Logic Trigger -> `toIntArgPacket` method on **IntArg** -> Network Packet -> Client-side UI Element (e.g., Slider)

