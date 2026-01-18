---
description: Architectural reference for ExplodeInteraction
---

# ExplodeInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.client
**Type:** Transient Data Object

## Definition
```java
// Signature
public class ExplodeInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The ExplodeInteraction class is a concrete implementation within the server's Interaction System. It represents a specific, data-driven behavior: causing an explosion. It is not a long-lived service, but rather a stateless configuration object that defines *how* an explosion should occur when an interaction is triggered.

Its primary architectural role is to act as an adapter between a generic game event and the specialized explosion logic. It is designed to be defined in external asset files (e.g., JSON) and loaded at runtime via its static CODEC.

Upon execution, its sole responsibility is to:
1.  Extract detailed contextual information from the provided InteractionContext, such as the precise impact location and the source of the interaction.
2.  Determine the appropriate Damage Source (e.g., distinguishing between a generic explosion and one caused by a specific projectile).
3.  Delegate the complex task of calculating and applying the explosion's effects (block destruction, entity damage) to the centralized ExplosionUtils service.

This design cleanly separates the *configuration* of an interaction from its *execution*, promoting reusability and data-driven design.

### Lifecycle & Ownership
- **Creation:** Instances are not created programmatically using the `new` keyword. They are deserialized from asset configuration files by the server's asset loading pipeline, which uses the static `CODEC` field for construction and validation. Each instance represents a reusable template for an explosion behavior.
- **Scope:** The object's lifecycle is tied to the server's asset registry. It persists in memory as a shared, immutable template as long as its defining asset is loaded.
- **Destruction:** The object is eligible for garbage collection when the asset registry is cleared or reloaded, typically during a server shutdown or a hot-reload of game assets.

## Internal State & Concurrency
- **State:** The class holds a single, critical piece of state: the `config` field, which is an instance of ExplosionConfig. This configuration is injected during deserialization via the CODEC and is considered **immutable** for the lifetime of the object. The `firstRun` method is otherwise stateless, operating exclusively on data provided in the InteractionContext.
- **Thread Safety:** This class is **not thread-safe** and is designed to operate exclusively on the main server thread. Its core method, `firstRun`, reads from and writes to shared world state (EntityStore, ChunkStore) through a CommandBuffer. Accessing this class from any other thread would bypass the server's transactional world update mechanisms, leading to race conditions, data corruption, and server instability.

## API Surface
The primary contract is the `firstRun` method, inherited from its parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(N) | Executes the explosion logic. Gathers context and delegates to ExplosionUtils. Complexity is proportional to the number of blocks and entities within the explosion radius. Throws an AssertionError if the internal ExplosionConfig is null. |
| CODEC | BuilderCodec | O(1) | A static factory used by the asset system to deserialize and construct ExplodeInteraction instances from configuration data. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly in procedural code. Instead, it is configured within a larger asset definition, such as an item or projectile, and is triggered automatically by the server's Interaction Module.

A conceptual asset configuration might look like this:

```yaml
# (Conceptual) assets/data/items/grenade.json
{
  "id": "hytale:grenade",
  "onImpact": {
    "type": "ExplodeInteraction",
    "Config": {
      "radius": 5.0,
      "blockDamage": 100.0,
      "entityDamage": 50.0,
      "breaksBlocks": true
    }
  }
}
```
When a grenade entity impacts a surface, the Interaction Module would identify the `onImpact` handler, find the corresponding ExplodeInteraction template, and invoke its `firstRun` method with the context of the impact event.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ExplodeInteraction()`. This bypasses the CODEC-based initialization, leaving the internal `config` field null. Any subsequent call to `firstRun` will result in a critical assertion failure.
- **Manual Invocation:** Do not call the `firstRun` method with a manually constructed InteractionContext. The context object is highly complex and is expected to be fully populated by the core engine. An incomplete context will lead to unpredictable behavior and NullPointerExceptions.
- **State Modification:** Do not attempt to modify the `config` field after initialization via reflection or other means. These objects are treated as immutable templates and may be shared across multiple concurrent interactions.

## Data Pipeline
The flow of data during an explosion event is unidirectional, passing from high-level game events down to low-level world state modifications.

> Flow:
> Game Event (e.g., Projectile Impact, Timed Detonation) -> Server Interaction Module -> **ExplodeInteraction.firstRun** -> Context Resolution (calculates precise position and damage source) -> `ExplosionUtils.performExplosion` -> CommandBuffer Population (queues block and entity changes) -> End of Tick -> CommandBuffer Flush -> World State Updated

