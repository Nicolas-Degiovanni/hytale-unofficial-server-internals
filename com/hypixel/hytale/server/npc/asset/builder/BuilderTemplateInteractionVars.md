---
description: Architectural reference for BuilderTemplateInteractionVars
---

# BuilderTemplateInteractionVars

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient

## Definition
```java
// Signature
public class BuilderTemplateInteractionVars extends BuilderCodecObjectHelper<Map<String, String>> {
```

## Architecture & Concepts
The BuilderTemplateInteractionVars is a specialized, configuration-driven component responsible for resolving the final set of variables used in an NPC interaction. It serves as the bridge between statically defined NPC templates (in JSON format) and the dynamic, runtime state of the game, encapsulated within an ExecutionContext.

Architecturally, this class is a concrete implementation within a larger asset building framework. Its inheritance from BuilderCodecObjectHelper signifies that it leverages the engine's standardized codec system for deserializing asset data. The core design pattern employed is the **Contextual Override**. The builder first loads a default map of key-value string pairs from a configuration file. At runtime, it attempts to retrieve a more specific, overriding map from the current ExecutionContext. This allows game designers to define baseline NPC behaviors in assets, while server logic can dynamically alter those behaviors based on player state, quest progress, or other in-game events.

The intentional disabling of the no-argument `build()` method is a critical design choice, forcing developers to provide a runtime context and preventing the accidental use of potentially incomplete or inappropriate template data.

### Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level NPC asset loading system when it encounters an `interactionVars` block within an NPC template file. It is not intended for direct instantiation by gameplay logic.
- **Scope:** Short-lived and task-specific. An instance of this builder exists only for the duration of a single NPC's asset resolution process. Once the `build(ExecutionContext)` method has been called and the resulting map is retrieved, the builder object has served its purpose and is eligible for garbage collection.
- **Destruction:** Managed by the Java garbage collector. No explicit cleanup methods are required or provided.

## Internal State & Concurrency
- **State:** The builder is **mutable**. Its internal state, an inherited field named `value` (a Map of String to String), is populated by the `readConfig` method. This state represents the default set of interaction variables loaded from the asset template.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. Its lifecycle is designed to be confined to a single-threaded asset loading or NPC instantiation sequence. Concurrent calls to `readConfig` and `build` would lead to unpredictable behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(data, extraInfo) | void | O(N) | Deserializes a JsonElement into the internal map of default variables. N is the number of variables. This is the primary data ingestion point. |
| build(context) | Map<String, String> | O(1) | Resolves the final interaction variables. Returns the map from the ExecutionContext if present, otherwise returns the default map loaded from config. |
| build() | Map<String, String> | N/A | **Unsupported.** Always throws UnsupportedOperationException to enforce context-aware usage. |

## Integration Patterns

### Standard Usage
The typical lifecycle involves an asset loader creating the builder, populating it from a JSON source, and a runtime system later using it to resolve the final variables with a specific context.

```java
// 1. In an asset loading system, the builder is created and configured
BuilderTemplateInteractionVars builder = new BuilderTemplateInteractionVars();
JsonElement interactionVarsJson = npcAsset.get("interactionVars");
builder.readConfig(interactionVarsJson, assetLoadingExtraInfo);

// ... time passes, an NPC interaction is triggered ...

// 2. In the NPC interaction logic, the final variables are resolved
ExecutionContext currentContext = createInteractionContext(player, npc);
Map<String, String> finalVars = builder.build(currentContext);

// 3. Use the resolved variables to drive interaction logic
if ("true".equals(finalVars.get("canTrade"))) {
    // Open trade UI
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not manually instantiate this class within gameplay code. It should be managed by the asset system to ensure it is correctly populated from NPC templates.
- **Calling parameter-less build:** Never call the `build()` method. It is explicitly disabled. Always provide an ExecutionContext to the `build(context)` method.
- **State Reuse:** Do not reuse a single builder instance for multiple different NPC templates. The internal state is populated by `readConfig` and is not cleared. A new builder must be created for each distinct configuration.

## Data Pipeline
The flow of data through this component involves two distinct inputs: a static configuration and a dynamic context, which are merged to produce a final output.

> Flow:
> NPC Template (JSON) -> Asset Deserializer -> **BuilderTemplateInteractionVars.readConfig()** -> [Internal State: Default Vars]
>
> Runtime Interaction -> ExecutionContext -> **BuilderTemplateInteractionVars.build(context)** -> Final `Map<String, String>` -> NPC Interaction System

