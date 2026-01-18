---
description: Architectural reference for HeadMotionSequence
---

# HeadMotionSequence

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility
**Type:** Transient Data Model

## Definition
```java
// Signature
public class HeadMotionSequence extends MotionSequence<HeadMotion> implements HeadMotion, IAnnotatedComponentCollection {
```

## Architecture & Concepts
The HeadMotionSequence class is a specialized data structure within the server-side NPC behavior system. It represents a concrete, ordered series of individual HeadMotion instructions that an NPC can execute. This class does not contain logic for executing the motions itself; rather, it serves as an immutable data container that is interpreted by higher-level AI systems, such as behavior trees or finite state machines.

Architecturally, it is a leaf node in the NPC asset composition tree. It is constructed during the server's asset loading phase from configuration files via its corresponding builder, BuilderHeadMotionSequence. By extending MotionSequence, it inherits the core machinery for managing a timed sequence of events. By implementing the HeadMotion interface, an entire sequence can be treated as a single, composite HeadMotion instruction, allowing for complex behaviors to be nested and reused. The IAnnotatedComponentCollection interface suggests it also carries metadata used by debugging tools or visual editors.

## Lifecycle & Ownership
- **Creation:** A HeadMotionSequence is never instantiated directly in game logic. It is exclusively constructed by the NPC asset pipeline via a BuilderHeadMotionSequence instance. This process occurs once when NPC definitions are loaded into memory, typically at server startup or during a hot-reload. The BuilderSupport parameter provides necessary context and shared resources from the asset loading environment.
- **Scope:** The object's lifetime is tied to the NPC asset definition it belongs to. It is effectively a static component of an NPC's behavioral template. Multiple NPC instances of the same type will share a reference to the same HeadMotionSequence instance.
- **Destruction:** The object is marked for garbage collection when its owning NPC asset definition is unloaded from the server. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The internal state consists of an ordered collection of HeadMotion steps. This state is provided during construction and is deeply **Immutable** thereafter. The sequence of motions cannot be altered at runtime.
- **Thread Safety:** As an immutable data object, HeadMotionSequence is inherently **Thread-Safe** for all read operations. It can be safely accessed by multiple NPC behavior threads simultaneously without locks or synchronization. The creation process itself is confined to the single-threaded asset loading pipeline.

## API Surface
The public contract of HeadMotionSequence is primarily defined by its parent class, MotionSequence, and the HeadMotion interface. The class itself exposes no unique public methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| HeadMotionSequence(builder, support) | constructor | O(N) | Internal constructor for asset pipeline. Complexity is proportional to the number of steps in the sequence. |
| *update(delta)* | *varies* | O(1) | **Inherited.** Advances the sequence playback. Managed by the NPC behavior system. |
| *getCurrentStep()* | *HeadMotion* | O(1) | **Inherited.** Returns the currently active HeadMotion instruction from the sequence. |
| *isFinished()* | *boolean* | O(1) | **Inherited.** Returns true if all motions in the sequence have completed. |
| *apply(npc)* | *void* | O(N) | **Inherited.** Applies the entire sequence as a single composite motion. |

*Note: Inherited symbols are italicized and represent the conceptual API provided by parent classes and interfaces.*

## Integration Patterns

### Standard Usage
A HeadMotionSequence is retrieved from an NPC's component data and passed to an executor, which is responsible for ticking the sequence over time and applying the resulting motion to the NPC entity.

```java
// Conceptual example of a behavior tree node executing the sequence
// Assume 'npc' is the target entity and 'blackboard' holds its state

// 1. Retrieve the pre-loaded sequence from the NPC's definition
HeadMotionSequence lookAround = npc.getComponent(HeadMotionSequences.LOOK_AROUND);

// 2. The behavior system ticks the sequence until it is finished
if (!lookAround.isFinished()) {
    lookAround.update(deltaTime);
    HeadMotion currentMotion = lookAround.getCurrentStep();
    
    // 3. Apply the current motion step to the NPC
    currentMotion.apply(npc);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new HeadMotionSequence()`. The constructor requires internal builder context that is only available during the asset loading phase. Doing so will fail and violates the design principle of separating data definition from game logic.
- **Runtime Modification:** Do not attempt to use reflection or other means to alter the internal list of motion steps after creation. These sequences are shared resources; modifying one instance would affect all NPCs of that type, leading to unpredictable and difficult-to-debug behavior.

## Data Pipeline
HeadMotionSequence is the output of the NPC asset definition pipeline. It serves as compiled, in-memory data for the live game simulation.

> Flow:
> NPC Definition (JSON/YAML) -> Asset Deserializer -> **BuilderHeadMotionSequence** -> **HeadMotionSequence** (Instance) -> NPC Behavior Executor -> NPC Entity State

