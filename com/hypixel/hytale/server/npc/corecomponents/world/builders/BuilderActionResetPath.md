---
description: Architectural reference for BuilderActionResetPath
---

# BuilderActionResetPath

**Package:** com.hypixel.hytale.server.npc.corecomponents.world.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderActionResetPath extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionResetPath class is a concrete implementation of the **Factory Method Pattern**, designed to construct runtime AI components from static asset definitions. It serves as a configuration-time blueprint for creating an ActionResetPath object, which is the executable instruction that an NPC's AI runtime will process.

This class is part of a larger, extensible system where numerous BuilderActionBase subclasses are registered. This architecture allows designers and developers to define new NPC behaviors without modifying the core AI execution engine. An external system, such as a behavior tree editor or an asset loader, discovers and utilizes these builders to translate a high-level NPC definition (e.g., from a JSON file) into a graph of executable Action objects.

In essence, BuilderActionResetPath is the bridge between the static *definition* of an action and its live, *instantiated* form within the game world.

### Lifecycle & Ownership
- **Creation:** Instantiated by the NPC asset loading pipeline when it encounters a "Reset Path" action type within an NPC's behavior definition asset. It is not created by the main game loop or by gameplay logic.
- **Scope:** Short-lived and transient. Its existence is typically confined to the asset deserialization and instantiation process. Once the `build` method is called and the resulting ActionResetPath object is integrated into the NPC's behavior tree, this builder instance is no longer required and can be garbage collected.
- **Destruction:** Eligible for garbage collection immediately after the NPC's behavior graph is fully constructed. Holding a reference to this object post-initialization is an architectural violation.

## Internal State & Concurrency
- **State:** **Immutable and Stateless**. This class contains no member fields and its behavior is entirely defined by its type. All method calls produce consistent results without side effects on the builder object itself.
- **Thread Safety:** **Fully Thread-Safe**. Due to its stateless nature, a single instance of BuilderActionResetPath can be safely shared across multiple threads to build ActionResetPath objects concurrently without the need for external locking or synchronization.

## API Surface
The public contract is minimal, focusing exclusively on metadata and object creation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Action | O(1) | Factory method. Constructs and returns a new ActionResetPath instance. |
| getShortDescription() | String | O(1) | Provides a human-readable summary for UI tooltips or editors. |
| getLongDescription() | String | O(1) | Provides a detailed description for UI tooltips or editors. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns a metadata flag indicating the production readiness of this action. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in standard gameplay code. It is invoked transparently by the server's NPC asset systems. The conceptual usage by the framework is as follows:

```java
// Hypothetical usage by an NPC Asset Loader
// The loader would have a map of action names to builder classes.
BuilderActionBase builder = npcActionRegistry.getBuilderFor("ResetPath");
Action runtimeAction = builder.build(builderSupportContext);
npcBehaviorTree.addAction(runtimeAction);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation for Logic:** Do not instantiate this class within an NPC's update tick or other gameplay logic. It is a configuration-time object, not a runtime service.
- **Caching Instances:** There is no performance benefit to caching instances of BuilderActionResetPath. The framework is designed to create them on-demand during asset loading.
- **Extending for Gameplay:** Do not extend this class to add gameplay logic. To create a new NPC action, you must create a new Action class and a corresponding Builder class for it.

## Data Pipeline
BuilderActionResetPath acts as a transformation step in the NPC initialization pipeline, converting declarative data into an executable object.

> Flow:
> NPC Behavior Asset (JSON) -> Asset Deserializer -> **BuilderActionResetPath.build()** -> ActionResetPath (Instance) -> NPC Behavior Tree -> AI Executor

