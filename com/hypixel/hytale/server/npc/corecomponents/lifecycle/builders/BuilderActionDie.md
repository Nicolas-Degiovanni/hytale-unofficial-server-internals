---
description: Architectural reference for BuilderActionDie
---

# BuilderActionDie

**Package:** com.hypixel.hytale.server.npc.corecomponents.lifecycle.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionDie extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionDie class is a configuration-time component within the Hytale NPC Behavior Asset system. It embodies the Builder design pattern, serving as a factory for creating instances of ActionDie, the concrete runtime command that terminates an NPC's existence.

Its primary role is to act as an intermediary between the declarative asset definition (e.g., a JSON file describing an NPC's behavior) and the instantiated, executable game object. The server's asset loading pipeline discovers and instantiates this builder when it parses a corresponding "die" action in an NPC behavior graph.

Methods such as getShortDescription and getBuilderDescriptorState do not influence runtime behavior. They provide essential metadata for design-time tools, such as a visual NPC behavior editor, allowing the tool to display human-readable information and stability warnings without needing to construct the final runtime action.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the server's NPC asset deserialization and loading system. This occurs when an NPC asset containing a "die" action is loaded into memory. Developers must not instantiate this class directly.
- **Scope:** Extremely short-lived and transient. An instance of BuilderActionDie exists only for the brief moment required to call its build method during the asset loading sequence.
- **Destruction:** The object is immediately eligible for garbage collection after the build method returns its ActionDie product. It is not registered in any service locator or context and holds no persistent state.

## Internal State & Concurrency
- **State:** BuilderActionDie is a stateless and immutable object. It contains no member fields and its behavior is fixed at compile time.
- **Thread Safety:** This class is inherently thread-safe. As a stateless factory, it can be safely used by multiple threads concurrently, although the asset loading pipeline is likely single-threaded per asset.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionDie | O(1) | Constructs and returns a new ActionDie instance. This is the primary factory method. |
| getShortDescription() | String | O(1) | Returns a brief, human-readable description for tooling and editors. |
| getLongDescription() | String | O(1) | Returns a detailed description, which in this case is identical to the short one. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns the stability status of this component for editor display. |

## Integration Patterns

### Standard Usage
A developer or designer does not interact with this class via Java code. Instead, they define its use declaratively within an NPC asset file. The engine's asset pipeline handles the instantiation and invocation.

A conceptual NPC behavior asset might look like this:
```json
// Example: my_npc.npc_behavior
{
  "behaviorTree": {
    "root": {
      "type": "Sequence",
      "children": [
        { "action": "WalkTo", "target": "Player" },
        { "action": "Die" } // This entry causes the engine to use BuilderActionDie
      ]
    }
  }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BuilderActionDie()` in game logic. The asset system is the sole owner of this class's lifecycle. Doing so bypasses the asset pipeline and creates unmanaged objects.
- **Runtime Invocation:** This builder is not intended for runtime use. To make an NPC die during gameplay, you must trigger the `ActionDie` object that this builder *creates*, typically by advancing the NPC's behavior tree to the correct node.

## Data Pipeline
This component operates entirely within the server's asset loading data pipeline. It does not process runtime game data.

> **Configuration Flow:**
> NPC Asset File (JSON) -> Server Asset Deserializer -> **BuilderActionDie.build()** -> ActionDie Instance -> Stored in NPC Behavior Tree Node

