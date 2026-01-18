---
description: Architectural reference for FloatArg
---

# FloatArg

**Package:** com.hypixel.hytale.server.core.asset.type.buildertool.config.args
**Type:** Transient

## Definition
```java
// Signature
public class FloatArg extends ToolArg<Float> {
```

## Architecture & Concepts
The FloatArg class is a server-side data model that defines the schema and constraints for a floating-point argument within the Builder Tools system. It is not a service or manager, but rather a configuration object that represents a single, configurable parameter of a tool, such as brush size or intensity.

Its role is threefold:
1.  **Configuration Schema:** Through its static CODEC, it defines how a float argument is deserialized from an asset definition file (e.g., a JSON file). This allows game designers to specify default values, minimums, and maximums declaratively.
2.  **Runtime Validation:** It provides logic to parse and validate user input, typically from a text command, ensuring the provided value respects the configured `min` and `max` boundaries.
3.  **Network Serialization:** It is responsible for converting its configuration into a specific network packet (BuilderToolFloatArg) to synchronize the tool's parameters with the game client, enabling the client-side UI to render appropriate controls like sliders.

This class acts as the bridge between a tool's static asset definition and its runtime behavior, enforcing server-authoritative constraints.

## Lifecycle & Ownership
-   **Creation:** Instances of FloatArg are not created directly by developers. They are instantiated and populated by the `BuilderCodec` during the server's asset loading phase when a Builder Tool definition is parsed.
-   **Scope:** The lifetime of a FloatArg instance is bound to the lifetime of the parent BuilderTool definition it belongs to. It persists in memory as long as the tool is registered and available on the server.
-   **Destruction:** The object is marked for garbage collection when its parent BuilderTool definition is unloaded, for example, during a server shutdown or a hot-reload of game assets.

## Internal State & Concurrency
-   **State:** FloatArg is a mutable data-holder. Its primary state consists of the inherited `value` field, along with its own `min` and `max` constraint fields. This state is populated once during deserialization and is intended to be treated as read-only thereafter.

-   **Thread Safety:** This class is **not thread-safe**. It is designed to be configured on the main server thread during asset loading. Subsequent reads may occur from game-loop threads, but concurrent modification of its `min` or `max` fields after initialization will lead to undefined behavior and is strictly unsupported.

## API Surface
The public API is focused on validation and network packet creation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fromString(String str) | Float | O(1) | Parses a string into a float and validates it against the configured min/max bounds. Throws ToolArgException if the value is out of range. |
| toFloatArgPacket() | BuilderToolFloatArg | O(1) | Constructs a new, network-ready Data Transfer Object (DTO) containing the default value, min, and max. |
| setupPacket(BuilderToolArg packet) | void | O(1) | Populates a generic parent packet with this argument's specific type and data payload. This is an implementation of the parent ToolArg contract. |

## Integration Patterns

### Standard Usage
FloatArg is not used directly in gameplay logic. Instead, it is defined declaratively within a tool's asset file. The system then uses the instance to validate input and synchronize with the client.

A conceptual example of the framework using a loaded FloatArg instance:
```java
// Assume 'tool' is a loaded BuilderTool and 'userInput' is "5.5"
// The 'sizeArg' was created by the CODEC from a config file.
FloatArg sizeArg = tool.getArgument("size", FloatArg.class);

try {
    // The framework validates the user's input string
    float validatedSize = sizeArg.fromString(userInput);

    // If successful, the validated value is used
    tool.applyEffect(validatedSize);

} catch (ToolArgException e) {
    // Handle the error, e.g., send a message to the player
    player.sendMessage(e.getLocalizedMessage());
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new FloatArg()`. The object's state is meaningless unless it is configured by the asset loading system via its `CODEC`. All tool arguments must be defined in data files.
-   **State Mutation After Load:** Do not modify the `min` or `max` fields after the server has finished loading assets. This can lead to desynchronization between the server's validation logic and the client's UI.
-   **Ignoring Validation:** Do not parse a float from a string manually and pass it to a tool. Always use the `fromString` method to ensure server-side constraints are enforced.

## Data Pipeline
The FloatArg class is a critical step in the data flow from tool configuration to client-side presentation and server-side execution.

> Flow:
> Tool Asset File (JSON) -> Server Asset Loader with `BuilderCodec` -> **FloatArg Instance** -> `setupPacket` Method -> `BuilderToolArg` Network Packet -> Game Client -> Client-Side UI (Slider) -> User Input (Command) -> Server Command Handler -> `fromString` Method -> Validated Float Value

