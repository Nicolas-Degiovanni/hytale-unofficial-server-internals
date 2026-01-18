---
description: Architectural reference for BuilderBodyMotionMatchLook
---

# BuilderBodyMotionMatchLook

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderBodyMotionMatchLook extends BuilderBodyMotionBase {
```

## Architecture & Concepts
The BuilderBodyMotionMatchLook class is a factory component within the server-side NPC behavior system. Its sole responsibility is to instantiate the BodyMotionMatchLook runtime component from a static asset definition. This class acts as a crucial link in the data-driven design of Hytale NPCs, translating a declarative configuration (typically from a JSON file) into a live, executable behavior object.

This builder represents one of the simplest forms of behavior: making an NPC's body rotate to match its current look direction. The lack of configurable parameters, evidenced by the empty `readConfig` method, indicates this is a fixed, atomic behavior that is either present or absent on an NPC. It is part of a larger composite system where complex NPC movements are constructed by combining multiple, single-responsibility motion components.

## Lifecycle & Ownership
- **Creation:** Instantiated by the NPC asset loading service when it parses an NPC definition file and encounters an entry for the "BodyMotionMatchLook" component. It is never created directly by game logic.
- **Scope:** Ephemeral. The builder's lifetime is confined to the instantiation of a single BodyMotionMatchLook component. It does not persist after the `build` method is called.
- **Destruction:** The object becomes eligible for garbage collection immediately after its `build` method returns its product. It holds no references and is not registered in any persistent context.

## Internal State & Concurrency
- **State:** This class is stateless and immutable. It contains no internal fields and its behavior is fixed. The `readConfig` method is a no-op, reinforcing that no external data alters its state.
- **Thread Safety:** Inherently thread-safe. Due to its stateless nature, a single instance could be used across multiple threads without risk of data corruption. However, the standard operational pattern is to create a new builder for each component instantiation.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | BodyMotionMatchLook | O(1) | Constructs and returns a new BodyMotionMatchLook instance. This is the primary factory method. |
| readConfig(JsonElement) | BuilderBodyMotionMatchLook | O(1) | Fulfills the configuration contract of its parent class. This implementation is a no-op as the component is not configurable. |
| getShortDescription() | String | O(1) | Provides a brief, human-readable description for tooling or debugging. |
| getBuilderDescriptorState() | BuilderDescriptorState | O(1) | Returns metadata indicating the production-readiness of this component. Returns **Stable**. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in game logic. It is invoked exclusively by the NPC asset pipeline. A game designer or developer would enable this behavior by including it in an NPC's JSON definition file.

**Conceptual Example (NPC JSON Definition):**
```json
{
  "name": "guard_npc",
  "components": [
    {
      "type": "BodyMotionMatchLook"
    }
  ]
}
```
The framework processes this entry by locating and using the BuilderBodyMotionMatchLook class to instantiate the corresponding runtime component.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new BuilderBodyMotionMatchLook()`. This class provides no value outside the context of the asset loading framework which supplies the necessary `BuilderSupport` context to its `build` method.
- **Manual Configuration:** Do not call `readConfig`. This method is part of an automated pipeline and has no effect for this specific builder.

## Data Pipeline
The flow of data from definition to runtime object is linear and unidirectional. This builder is a single transformation step in that pipeline.

> Flow:
> NPC Definition (JSON) -> Asset Parsing Service -> **BuilderBodyMotionMatchLook** -> BodyMotionMatchLook (Runtime Instance) -> NPC Component Registry

