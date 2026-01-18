---
description: Architectural reference for BuilderActionCombatAbility
---

# BuilderActionCombatAbility

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator.corecomponents.builders
**Type:** Factory

## Definition
```java
// Signature
public class BuilderActionCombatAbility extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionCombatAbility class is a factory component within the server-side NPC AI framework. It serves a single, critical purpose: to instantiate an ActionCombatAbility object at runtime. This class embodies the Builder pattern as applied to Hytale's entity behavior system.

In Hytale's architecture, NPC behaviors are defined declaratively in asset files (e.g., JSON). The engine parses these assets and uses corresponding "Builder" classes to translate the data definitions into live, executable action objects. BuilderActionCombatAbility is the concrete implementation for the "start a combat ability" action.

It acts as a bridge between the static NPC asset definition and the dynamic, in-world execution of an NPC's behavior tree. When an NPC's AI decides to use a combat ability, the behavior system locates this builder and invokes its build method to create the stateful action object that will be executed by the game loop.

### Lifecycle & Ownership
- **Creation:** Instances of BuilderActionCombatAbility are not created directly by game logic developers. They are instantiated by the Hytale server's asset management and behavior systems during the loading of NPC configuration files. The engine maintains a registry of these builders, mapping them to specific action types defined in assets.

- **Scope:** A builder instance is typically stateless and reusable. It may persist for the lifetime of the server process, cached in a central registry, or it may be created on-demand when an NPC's behavior asset is loaded. The object it *produces*, ActionCombatAbility, has a much shorter lifecycle, tied to the execution of a single action within an NPC's behavior tree.

- **Destruction:** Managed by the Java Garbage Collector. Instances are eligible for collection when the corresponding NPC asset definitions are unloaded from the server's asset registry.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no member fields and its behavior is entirely defined by its method implementations. Its sole purpose is to construct other objects.

- **Thread Safety:** As a stateless object, BuilderActionCombatAbility is inherently **thread-safe**. It can be safely shared and its build method can be invoked from multiple threads without synchronization, provided the supplied BuilderSupport context is thread-safe or confined to the appropriate thread.

## API Surface
The public contract is minimal, focusing on object creation and metadata for tooling.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionCombatAbility | O(1) | Factory method. Instantiates and returns a new ActionCombatAbility. |
| getShortDescription() | String | O(1) | Provides a brief, human-readable description for use in development tools. |
| getLongDescription() | String | O(1) | Provides a detailed, human-readable description. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns metadata indicating the production-readiness of this component. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class directly in Java code. Instead, they declare its use within an NPC's behavior asset file. The engine handles the lookup and invocation internally. The conceptual flow for the engine is as follows:

```java
// Engine-level code (conceptual)
// 1. An NPC asset is loaded, and this builder is registered.
BuilderActionBase builder = behaviorRegistry.get("ActionCombatAbility");

// 2. At runtime, the NPC's AI needs to create the action.
BuilderSupport context = createSupportForNpc(npc);
ActionCombatAbility action = ((BuilderActionCombatAbility) builder).build(context);

// 3. The engine executes the newly created action.
npc.getBehaviorController().execute(action);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new BuilderActionCombatAbility()`. The behavior system relies on its own managed instances, which are registered during asset loading. Direct creation bypasses this system and will lead to unmanaged, non-functional AI components.
- **Stateful Extension:** Do not extend this class to add state. The builder pattern in this context assumes statelessness for reusability and thread safety. Adding state will break this contract and cause unpredictable behavior, especially in a multi-threaded server environment.

## Data Pipeline
This class functions as a transformation point in the NPC behavior pipeline, converting declarative data into an executable object.

> Flow:
> NPC Asset File (JSON) -> Asset Loader -> **BuilderActionCombatAbility** -> `build()` Invocation -> ActionCombatAbility Instance -> NPC Behavior Executor

