---
description: Architectural reference for BuilderActionWithDelay
---

# BuilderActionWithDelay

**Package:** com.hypixel.hytale.server.npc.corecomponents.builders
**Type:** Transient

## Definition
```java
// Signature
public abstract class BuilderActionWithDelay extends BuilderActionBase {
```

## Architecture & Concepts
BuilderActionWithDelay is an abstract base class that serves as a foundational component within the server-side NPC asset pipeline. It embodies the **Template Method** design pattern, providing a standardized structure for parsing and handling time-based delays for NPC actions defined in JSON assets.

Its primary architectural role is to abstract and encapsulate the common logic of reading a "Delay" property. This property is typically defined as a range (e.g., `[1.5, 3.0]`) representing the minimum and maximum time in seconds an NPC should wait before or after performing an action. By inheriting from this class, concrete action builders (like a builder for a "Wait" or "Taunt" action) can gain delay-parsing functionality without duplicating code.

This class acts as an intermediary between the raw JSON configuration and the final, executable in-memory Action object used by the NPC's AI runtime. It ensures that all delay-related configurations are parsed, validated, and stored consistently across different action types.

### Lifecycle & Ownership
- **Creation:** Instances of concrete subclasses are created by a central factory or registry within the NPC asset loading system. This typically occurs when the system parses an NPC definition file and encounters a JSON object representing an action that requires a delay. This class itself, being abstract, is never instantiated directly.
- **Scope:** The lifecycle of a builder instance is extremely short and confined to a single parsing operation. It is a **transient object** that exists only to transform a specific JSON element into a corresponding Action object.
- **Destruction:** Once the builder's `build` method is called and the final Action object is returned, the builder instance has fulfilled its purpose. It holds no further references and becomes eligible for garbage collection.

## Internal State & Concurrency
- **State:** BuilderActionWithDelay is **stateful and mutable**. Its primary state is the `delayRange` field, which is populated by the `readCommonConfig` method during the parsing process. This state is specific to the single JSON object being processed.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. The asset loading pipeline is expected to create a new builder instance for each asset it processes. Attempting to reuse a single instance to parse multiple JSON objects, especially concurrently, will result in race conditions and corrupted state within the `delayRange` field.

## API Surface
The public contract is designed for use by the asset building framework and concrete subclasses.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readCommonConfig(JsonElement) | Builder<Action> | O(1) | Parses the "Delay" key from the provided JSON. Populates internal state. Delegates to the superclass for other common properties. |
| getDelayRange(BuilderSupport) | double[] | O(1) | Retrieves the parsed and validated delay range. Requires a BuilderSupport context to resolve any dynamic values. |
| getDefaultTimeoutRange() | double[] | O(1) | A protected template method that provides the default delay value if one is not specified in the JSON. Subclasses may override this. |

## Integration Patterns

### Standard Usage
This class is not used directly. A developer creating a new, delay-able NPC action must extend it and implement the `build` method. The framework handles instantiation and invocation.

```java
// A concrete builder for a "Wait" action
public class BuilderWaitAction extends BuilderActionWithDelay {

    // The framework calls this after readCommonConfig
    @Override
    public Action build(BuilderSupport support) {
        // Retrieve the delay parsed by the parent class
        double[] parsedDelay = this.getDelayRange(support);

        // Use the parsed value to construct the final Action object
        return new WaitAction(parsedDelay);
    }

    // Optionally, override the default
    @Override
    protected double[] getDefaultTimeoutRange() {
        return new double[]{ 0.5, 0.5 }; // A shorter default wait
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Never reuse a builder instance. The internal state is tied to a single parse operation. Always allow the asset framework to create a fresh instance for each JSON object.
- **Manual Invocation:** Do not call `readCommonConfig` or other lifecycle methods manually. The asset loading system is responsible for invoking the builder's methods in the correct order. Bypassing the framework can lead to partially initialized or invalid Action objects.

## Data Pipeline
This class functions as a specific processing stage within the larger NPC asset-to-runtime data pipeline. It is responsible for handling one specific piece of data (the delay) during the transformation.

> Flow:
> NPC Asset JSON File -> Asset Loading Service -> Builder Factory -> **BuilderActionWithDelay** (via subclass) -> In-Memory `Action` Object -> NPC AI State Machine Execution

