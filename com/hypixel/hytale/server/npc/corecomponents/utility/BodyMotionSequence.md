---
description: Architectural reference for BodyMotionSequence
---

# BodyMotionSequence

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility
**Type:** Component / Data Model

## Definition
```java
// Signature
public class BodyMotionSequence extends MotionSequence<BodyMotion> implements BodyMotion, IAnnotatedComponentCollection {
```

## Architecture & Concepts
The BodyMotionSequence is a composite component within the server-side NPC AI framework. It orchestrates a series of individual BodyMotion objects, executing them in a predefined order. This allows designers to create complex, multi-stage behaviors—such as a patrol route or a specific combat maneuver—that can be treated as a single, atomic operation by the higher-level AI.

Architecturally, it serves two primary roles:
1.  **A State Machine:** It internally manages the progression through a list of child motions. Its state is defined by which child motion is currently active.
2.  **A Façade:** By implementing the BodyMotion interface itself, it exposes the *currently active* child's behavior to the rest of the NPC system. This allows systems like navigation to interact with the sequence transparently, without needing to know which specific step is currently executing.

This component is fundamental for creating scripted NPC behaviors that unfold over time, bridging the gap between low-level motion primitives and high-level AI goals.

### Lifecycle & Ownership
-   **Creation:** A BodyMotionSequence is never instantiated directly with `new`. It is exclusively constructed by a `BuilderBodyMotionSequence` during the server's asset loading phase. This process parses a declarative format (e.g., JSON) that defines the sequence of motions, ensuring the object is always created in a valid and complete state.
-   **Scope:** The lifetime of a BodyMotionSequence is tightly coupled to the NPC behavior that requires it. It is typically a member of a specific AI state object. It persists only as long as the NPC remains in that state.
-   **Destruction:** The object is marked for garbage collection when the NPC's AI transitions to a new state and the reference to the sequence is dropped. It holds no global state and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** This class is highly stateful and mutable. Its primary state, which includes the list of all motion steps and a reference to the `activeMotion`, is inherited from its parent, `MotionSequence`. The state changes on each tick as the sequence progresses from one motion to the next.
-   **Thread Safety:** This class is **not thread-safe** and must not be accessed from multiple threads. It is designed to be owned and manipulated exclusively by the single thread responsible for updating the corresponding NPC's AI and physics. Unsynchronized, concurrent access would lead to severe race conditions when updating the active motion.

## API Surface
The public contract is minimal, as most of the complex logic for advancing the sequence is handled by an inherited update method (not shown). The primary interaction point is for querying the current state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSteeringMotion() | BodyMotion | O(1) | Delegates to the currently active motion step to retrieve its steering component. Returns null if the sequence is complete or has not yet started. |

**Warning:** Callers must be prepared to handle a null return from `getSteeringMotion`. A null value indicates that no steering should be applied for the current tick, which is a valid state (e.g., during a pause or at the end of the sequence).

## Integration Patterns

### Standard Usage
The BodyMotionSequence is held and updated by an NPC's AI controller. On each server tick, the navigation system queries it to determine how to move the NPC.

```java
// Pseudocode within an NPC's AI update method

// The sequence is a member of the current AI state
BodyMotionSequence currentSequence = this.getActiveBehaviorSequence();

// 1. The AI system first ticks the sequence to advance its internal state
currentSequence.update(deltaTime);

// 2. The navigation system then queries the result
BodyMotion steering = currentSequence.getSteeringMotion();

// 3. The result is applied to the NPC's physics controller
if (steering != null) {
    npc.getNavigationController().applySteering(steering);
}
```

### Anti-Patterns (Do NOT do this)
-   **Manual Construction:** Never attempt to construct this class manually. The internal list of motion steps can only be populated correctly via the `BuilderBodyMotionSequence` during asset loading.
-   **External State Modification:** Do not attempt to modify the internal state of the sequence (like the active motion) from outside the class. The sequence's progression logic is self-contained and must not be interfered with.
-   **Cross-Thread Access:** Never share an instance of BodyMotionSequence between different NPC update threads. Each instance is owned by a single NPC.

## Data Pipeline
The BodyMotionSequence is the runtime representation of data that originates in an asset file.

> Flow:
> NPC Behavior Asset (JSON) -> `BuilderBodyMotionSequence` -> **BodyMotionSequence Instance** -> NPC AI State -> Navigation System -> NPC Physics Update

