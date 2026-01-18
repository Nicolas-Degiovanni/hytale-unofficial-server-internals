---
description: Architectural reference for BuilderActionDisplayName
---

# BuilderActionDisplayName

**Package:** com.hypixel.hytale.server.npc.corecomponents.audiovisual.builders
**Type:** Transient Configuration Object

## Definition
```java
// Signature
public class BuilderActionDisplayName extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionDisplayName class is a server-side component within the NPC asset definition framework. It serves as a configuration blueprint, not a runtime component. Its primary role is to deserialize a fragment of an NPC's JSON definition that specifies the display name action.

This class embodies a key architectural principle of the NPC system: the separation of static asset data from dynamic runtime behavior. It achieves this through its internal use of a StringHolder. Instead of storing a raw string, it stores a container that can resolve the final string value at runtime via an ExecutionContext provided by the BuilderSupport. This allows for dynamic names based on game state, player language, or other contextual factors, even though the initial configuration is static.

This builder is one of many `BuilderActionBase` implementations that are discovered and instantiated by a higher-level asset parser. After configuration via `readConfig`, its `build` method is invoked to produce the final, immutable `ActionDisplayName` object which is attached to the NPC's behavior tree.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the central NPC asset deserializer when an "ActionDisplayName" component is declared within an NPC's JSON configuration file. The `readConfig` method is called immediately following instantiation to populate the builder's state from the JSON data.
-   **Scope:** The builder's lifecycle is tied directly to the lifecycle of the runtime `ActionDisplayName` it produces. The `build` method passes a reference of the builder itself (`this`) to the `ActionDisplayName` constructor. Therefore, the builder object persists in memory as long as the resulting action exists on an NPC.
-   **Destruction:** The builder is eligible for garbage collection only when the `ActionDisplayName` it created is destroyed, which typically occurs when an NPC is unloaded or its behavior profile is completely redefined.

## Internal State & Concurrency
-   **State:** The internal state is mutable during the initial asset loading and configuration phase via the `readConfig` method. The primary state is the `displayName` StringHolder. After the `build` method is called, the builder's state should be considered effectively immutable. Modifying it post-build is an anti-pattern that results in undefined behavior.
-   **Thread Safety:** This class is **not thread-safe**. All configuration and build operations are expected to occur synchronously on the main server thread during the asset loading pipeline. Concurrent calls to `readConfig` or `getDisplayName` will lead to race conditions and unpredictable state.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionDisplayName | O(1) | Constructs the final runtime ActionDisplayName component. |
| readConfig(JsonElement) | BuilderActionDisplayName | O(1) | Deserializes JSON data into the internal StringHolder. Returns `this` for chaining. |
| getDisplayName(BuilderSupport) | String | O(K) | Resolves the final string value at runtime. K is the complexity of the ExecutionContext lookup. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is exclusively manipulated by the NPC asset loading system. The typical sequence is managed internally by the asset parser.

```java
// Conceptual example of internal asset system usage
BuilderActionDisplayName builder = new BuilderActionDisplayName();
builder.readConfig(npcJson.get("displayNameAction"));

// The builder is then used to create the runtime component
ActionDisplayName action = builder.build(builderSupport);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never manually instantiate this class with `new`. The asset system is responsible for the entire lifecycle. Manual creation bypasses the configuration pipeline and will result in non-functional components.
-   **State Mutation Post-Build:** Do not call `readConfig` or otherwise attempt to modify the builder's state after the `build` method has been invoked. The resulting `ActionDisplayName` holds a live reference to the builder, and late-stage mutations can cause unpredictable name changes on active NPCs.

## Data Pipeline
The flow of data begins with a raw asset file and ends with a runtime component capable of resolving a dynamic string.

> Flow:
> NPC Definition JSON -> Asset Deserializer -> **BuilderActionDisplayName.readConfig()** -> **BuilderActionDisplayName.build()** -> ActionDisplayName (Runtime Component) -> NPC Behavior Engine

