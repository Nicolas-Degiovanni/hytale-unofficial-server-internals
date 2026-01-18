---
description: Architectural reference for AOECylinderSelector
---

# AOECylinderSelector

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.selector
**Type:** Configuration Object

## Definition
```java
// Signature
public class AOECylinderSelector extends AOECircleSelector {
```

## Architecture & Concepts
The AOECylinderSelector is a data-driven configuration class that defines a three-dimensional cylindrical volume for selecting entities and blocks. It serves as a fundamental building block within the server's Interaction System, allowing game designers to specify areas of effect for abilities, spells, or environmental triggers without writing new code.

Architecturally, this class embodies the **Factory Pattern**. The AOECylinderSelector instance itself holds the static configuration parameters (radius and height), which are deserialized from game data files via its static CODEC. Its primary responsibility is to create executable **Selector** instances through the newSelector method. This cleanly separates the immutable configuration from the transient, per-use runtime logic, which is encapsulated in the private inner class RuntimeSelector.

This class extends AOECircleSelector, inheriting the planar radius (range) and offset properties, and adds a vertical **height** component to form the cylinder. This inheritance promotes reuse and a consistent configuration structure for related geometric selectors.

## Lifecycle & Ownership
- **Creation:** AOECylinderSelector instances are not created directly using the new keyword in game logic. They are instantiated by the engine's **Codec** system during the server's asset loading phase. The system reads a definition from a data file (e.g., a JSON file defining a player ability) and uses the registered CODEC to construct the object.
- **Scope:** An instance persists for the lifetime of its containing configuration. For example, if it defines the area of effect for a specific monster's attack, it will remain in memory as long as that monster's definition is loaded. It is effectively a stateless, reusable template.
- **Destruction:** The object is eligible for garbage collection when its parent configuration is unloaded, such as during a server shutdown or a transition between worlds or game modes that use different asset sets.

## Internal State & Concurrency
- **State:** The AOECylinderSelector is **effectively immutable** after its creation by the Codec system. Its fields, such as height and the inherited range, are intended to be read-only configuration values. The inner RuntimeSelector class is stateless between invocations; it does not retain data across different game ticks or calls.
- **Thread Safety:** The outer AOECylinderSelector is thread-safe due to its immutable nature. However, the Selector instance produced by newSelector is **not thread-safe** and is designed exclusively for use within the server's main game loop. Its methods operate on a CommandBuffer, a construct central to Hytale's Entity Component System (ECS) that ensures all world modifications are queued and executed from a single thread, preventing race conditions.

**WARNING:** Never share a Selector instance across multiple threads or attempt to call its methods from outside the main server tick.

## API Surface
The primary contract is defined by the parent Selector interface, which is implemented by the internal RuntimeSelector.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| newSelector() | Selector | O(1) | Factory method. Returns a new runtime instance for performing the selection logic. |
| toPacket() | com.hypixel.hytale.protocol.Selector | O(1) | Serializes the selector's configuration into a network packet for client-side prediction or visualization. |
| *selectTargetEntities(...)* | void | O(N) | *On Selector*. Finds all valid entities within the cylinder. N is the number of entities in the coarse bounding box. |
| *selectTargetBlocks(...)* | void | O(R² * H) | *On Selector*. Finds all blocks within the cylinder. R is range, H is height. |

## Integration Patterns

### Standard Usage
Developers typically do not interact with this class directly in Java code. Instead, it is defined declaratively in data files. The system then retrieves and uses the configured selector as part of a larger interaction. The following conceptual example shows how the *engine* would use a pre-loaded selector.

```java
// Assume 'interaction' is an object loaded from configuration
// which contains a configured AOECylinderSelector.

Selector selector = interaction.getSelector().newSelector();

// The system provides a callback to process any entities found.
BiConsumer<Ref<EntityStore>, Vector4d> onEntityFound = (entityRef, sourcePosition) -> {
    // Queue a damage command for the found entity.
    DamageCommand.inflict(commandBuffer, entityRef, 10.0);
};

// Execute the selection logic against the current world state.
selector.selectTargetEntities(commandBuffer, sourceEntityRef, onEntityFound, null);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new AOECylinderSelector()`. The object is not designed for manual construction; it must be configured and loaded via the engine's Codec system to ensure proper initialization.
- **State Modification:** Do not attempt to modify the public fields like `height` or `range` at runtime. These are considered immutable after loading. Modifying them can lead to unpredictable behavior and desynchronization between the server and client.
- **Reusing RuntimeSelector:** Do not cache and reuse the `Selector` instance returned by `newSelector()` across multiple independent interactions. While it is stateless, the pattern is to request a new instance for each top-level interaction execution.

## Data Pipeline
The AOECylinderSelector functions as a spatial filter within a larger command and event processing pipeline. Its primary role is to translate a logical area of effect into a concrete list of targets.

> Flow:
> Game Event (e.g., Ability Cast) -> Interaction System -> **AOECylinderSelector.newSelector()** -> RuntimeSelector.selectTargetEntities -> TargetUtil queries EntityStore for a broad-phase box -> Fine-phase cylinder check and filtering (isDead, isInvulnerable) -> Valid targets passed to BiConsumer callback -> Callback queues commands (e.g., Damage) on the CommandBuffer

