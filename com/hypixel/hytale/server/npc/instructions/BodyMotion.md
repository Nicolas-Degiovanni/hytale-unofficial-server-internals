---
description: Architectural reference for BodyMotion
---

# BodyMotion

**Package:** com.hypixel.hytale.server.npc.instructions
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface BodyMotion extends Motion {
```

## Architecture & Concepts
The BodyMotion interface represents a specialized contract within the server-side NPC instruction system. It extends the base Motion interface, signifying that it describes a form of movement or physical action for a non-player character.

Its primary architectural role is to decouple high-level, complex animations from low-level, continuous steering behaviors. For example, an NPC might execute a *PlayAnimation* instruction that is also a BodyMotion. While the animation plays (e.g., a taunt), the NPC might still need to subtly turn to face a target. The BodyMotion interface provides the formal mechanism to retrieve this underlying steering component via the getSteeringMotion method.

This allows the NPC processing loop to handle layered behaviors gracefully. The main animation system can process the full BodyMotion, while the pathfinding and physics systems can query for the simpler, subordinate steering motion to guide the NPC's root position and orientation. By default, a BodyMotion considers itself its own steering motion, but this can be overridden to create composite instructions.

## Lifecycle & Ownership
As an interface, BodyMotion itself has no lifecycle. The lifecycle pertains to the concrete classes that implement it.

- **Creation:** Concrete BodyMotion implementations are typically instantiated by nodes within an NPC's Behavior Tree or a similar AI state machine. They are created on-demand in response to changing world states or AI objectives (e.g., `new WalkToTargetMotion(target)`).
- **Scope:** The scope of a BodyMotion instance is almost always transient. It exists only for the duration it is being actively processed by an NPC. Once the motion is completed, interrupted, or replaced, the object is dereferenced and becomes eligible for garbage collection.
- **Destruction:** There is no explicit destruction method. Cleanup is managed by the Java garbage collector after the NPC's instruction processor discards the reference.

## Internal State & Concurrency
- **State:** The interface itself is stateless. Concrete implementations are expected to be stateful, containing data such as target coordinates, animation names, or movement speeds. This state is generally considered mutable during the instruction's execution.
- **Thread Safety:** The contract is inherently thread-safe. However, implementations are **not guaranteed** to be. All NPC instruction processing is expected to occur on the main server thread for the world in which the NPC exists. Accessing or modifying a BodyMotion instance from an asynchronous task or different thread is an unsupported and dangerous operation that will lead to race conditions.

## API Surface
The public contract is minimal, focusing on its role as a provider for a steering component.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSteeringMotion() | BodyMotion | O(1) | Retrieves the underlying motion responsible for steering and pathfinding. May return null if no steering is applicable. May return *this* if the motion is its own steering component. |

## Integration Patterns

### Standard Usage
The primary consumer of this interface is the server's NPC update loop. The system retrieves the current instruction for an NPC, checks if it implements BodyMotion, and if so, uses it to influence the NPC's physics and orientation.

```java
// Simplified example from within an NPC update method
Motion currentMotion = npc.getActiveMotion();

if (currentMotion instanceof BodyMotion) {
    BodyMotion steering = ((BodyMotion) currentMotion).getSteeringMotion();
    if (steering != null) {
        // Pass the simplified steering motion to the pathfinding or physics system
        pathfindingSystem.applySteering(npc, steering);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Nulls:** Do not assume getSteeringMotion will always return a non-null value. A motion may be purely cosmetic (e.g., a particle effect) and have no steering component. Always perform a null check.
- **Casting to Concrete Types:** Avoid casting a BodyMotion to a specific implementation like WalkToTargetMotion. The system is designed to operate on the abstraction provided by the interface. Casting breaks this contract and makes the system brittle.

## Data Pipeline
BodyMotion acts as a control signal within the NPC AI pipeline, translating a high-level goal into a low-level steering instruction.

> Flow:
> Behavior Tree Node -> Instantiates **ConcreteBodyMotion** -> NPC Instruction Queue -> NPC Update Tick -> `getSteeringMotion()` is called -> Pathfinding System -> Physics Engine Update

