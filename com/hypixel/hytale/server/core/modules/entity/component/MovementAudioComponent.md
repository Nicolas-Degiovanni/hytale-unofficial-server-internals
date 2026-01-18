---
description: Architectural reference for MovementAudioComponent
---

# MovementAudioComponent

**Package:** com.hypixel.hytale.server.core.modules.entity.component
**Type:** Transient Component

## Definition
```java
// Signature
public class MovementAudioComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The MovementAudioComponent is a server-side, state-holding component within the Entity-Component-System (ECS) architecture. It does not directly produce audio. Instead, it serves as a data container that audio-related systems query to determine *if* and *what* movement sounds an entity should generate.

Its primary responsibilities are:
1.  **Cooldown Management:** It enforces a delay between repeatable movement sounds (like footsteps) to prevent audio spam. This is managed by the `nextMoveInRepeat` timer.
2.  **Contextual State:** It stores the `lastInsideBlockTypeId`, allowing systems to select appropriate audio clips based on the surface the entity is interacting with (e.g., grass, stone, wood).
3.  **Audience Filtering:** It provides a `ShouldHearPredicate` to efficiently determine which other entities are eligible to hear the sound, typically excluding the entity that produced it.

This component is a passive data structure; its state is read and mutated exclusively by server-side systems, such as a hypothetical `MovementAudioSystem`, during the main game tick.

## Lifecycle & Ownership
- **Creation:** An instance of MovementAudioComponent is attached to an Entity when it is spawned, provided its archetype is defined to have one. The `clone` method is invoked by the ECS framework when an entity is duplicated, ensuring the new entity receives a fresh component with default state.
- **Scope:** The component's lifetime is strictly bound to the Entity it is attached to. It persists as long as the parent Entity exists in the world.
- **Destruction:** The component is marked for garbage collection when its parent Entity is despawned or when the component is explicitly removed from the entity via the ECS API.

## Internal State & Concurrency
- **State:** The component is highly mutable. Its core fields, `lastInsideBlockTypeId` and `nextMoveInRepeat`, are designed to be updated frequently by game logic systems every tick. It holds no long-term or cached data beyond this frame-to-frame state.
- **Thread Safety:** **This component is not thread-safe.** It is designed to be accessed and modified only by the main server game loop thread. Unsynchronized access from other threads (e.g., networking or physics worker threads) will lead to race conditions, inconsistent state, and server instability. All interactions with this component must be marshaled to the main tick.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally registered type identifier for this component class. |
| getShouldHearPredicate(Ref) | ShouldHearPredicate | O(1) | Returns a pre-configured predicate to filter which entities should receive the audio event. |
| tickMoveInRepeat(float dt) | boolean | O(1) | Advances the internal cooldown timer. Returns true if the cooldown has expired. |
| setLastInsideBlockTypeId(int) | void | O(1) | Mutates the block type ID the entity was last inside. |
| setNextMoveInRepeat(float) | void | O(1) | Resets the movement sound cooldown timer to a new duration. |
| clone() | Component | O(1) | Creates a new instance with default values. For internal ECS use only. |

## Integration Patterns

### Standard Usage
This component is intended to be managed by a dedicated server system. The system retrieves the component from an entity, checks its state, and updates it as part of a larger gameplay logic flow.

```java
// In a hypothetical MovementAudioSystem processing an entity
MovementAudioComponent audioState = entity.getComponent(MovementAudioComponent.class);

if (audioState != null && entity.hasMovedThisTick()) {
    // tickMoveInRepeat returns true when the timer hits zero
    if (audioState.tickMoveInRepeat(deltaTime)) {
        // Logic to determine the sound to play based on block type
        int blockId = audioState.getLastInsideBlockTypeId();
        SoundEvent sound = getSoundForBlock(blockId);

        // Dispatch the sound event to nearby players
        World.sendSoundEvent(sound, entity.getPosition(), audioState.getShouldHearPredicate(entity.getRef()));

        // Reset the cooldown
        float cooldown = sound.getRepeatDelay();
        audioState.setNextMoveInRepeat(cooldown);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new MovementAudioComponent()`. Components must be added to entities via the entity manager (e.g., `entity.addComponent(MovementAudioComponent.class)`). The `clone` method is reserved for the engine's internal entity duplication process.
- **State Management Outside a System:** Do not have disparate parts of the codebase modifying this component's state. All mutations should be centralized within a single, authoritative system to ensure predictable behavior and prevent conflicting updates.
- **Ignoring the Cooldown:** Do not trigger sounds without first checking and updating the cooldown via `tickMoveInRepeat`. Bypassing this mechanism will result in audio spam and a poor player experience.

## Data Pipeline
The MovementAudioComponent acts as a stateful gate in the audio generation pipeline. It does not process data itself but enables other systems to make decisions.

> Flow:
> Entity Movement Detected -> `MovementAudioSystem` reads entity position -> System queries **MovementAudioComponent** -> If `tickMoveInRepeat` is true, a `SoundEvent` is created -> `SoundEvent` is dispatched to the network layer with the `ShouldHearPredicate` for filtering -> System updates **MovementAudioComponent** state (`setNextMoveInRepeat`).

