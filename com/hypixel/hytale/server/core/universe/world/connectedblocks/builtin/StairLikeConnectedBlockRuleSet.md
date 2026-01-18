---
description: Architectural reference for StairLikeConnectedBlockRuleSet
---

# StairLikeConnectedBlockRuleSet

**Package:** com.hypixel.hytale.server.core.universe.world.connectedblocks.builtin
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface StairLikeConnectedBlockRuleSet {
```

## Architecture & Concepts
The StairLikeConnectedBlockRuleSet interface defines a formal contract for any block that exhibits stair-like connection behavior. It is a fundamental component of the server's **Connected Blocks System**, which is responsible for dynamically altering a block's state and visual appearance based on its adjacent blocks.

This interface specifically standardizes the logic for determining the shape of a stair block. This includes resolving its state into straight sections, inner corners, and outer corners. By abstracting this logic into a common interface, the world engine can process any stair-like block polymorphically, without needing to know its specific material or type. The ruleset is queried by the world engine during block placement, block updates, or chunk generation to calculate the final, correct block state.

## Lifecycle & Ownership
As an interface, StairLikeConnectedBlockRuleSet does not have a lifecycle of its own. Instead, it defines the expected behavior for implementing classes.

- **Creation:** Concrete implementations of this interface are typically instantiated once at server startup by the Block Registry or a similar bootstrap manager. They are registered as part of a specific block's definition.
- **Scope:** The ruleset objects are designed to be stateless and immutable. They exist for the entire lifetime of the server session.
- **Destruction:** The rulesets are destroyed only when the server shuts down and the Java Virtual Machine garbage collects the class loaders.

**WARNING:** Implementations of this interface are expected to be stateless singletons. Storing mutable state within a ruleset implementation is a severe anti-pattern that will lead to concurrency violations and unpredictable block behavior across the world.

## Internal State & Concurrency
- **State:** This interface is stateless by definition. Concrete implementations must be immutable. All calculations performed by its methods should be pure functions, depending only on the input arguments.
- **Thread Safety:** Implementations must be unconditionally thread-safe. The Connected Blocks System will invoke methods on this interface from multiple worker threads simultaneously, especially during parallel chunk generation. Any implementation that is not thread-safe will cause world corruption.

## API Surface
The public contract is minimal, focusing exclusively on resolving stair shape and identifying the base material.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getStairType(int) | StairConnectedBlockRuleSet.StairType | O(1) | Calculates the appropriate stair shape based on a bitmask representing neighboring blocks. |
| getMaterialName() | String | O(1) | Returns the name of the base material for this ruleset. May be null. Used for sounds and effects. |

## Integration Patterns

### Standard Usage
A developer does not typically interact with this interface directly. The world engine's Connected Blocks System identifies a block that uses this ruleset and invokes its methods automatically during a block state update. The system is responsible for calculating the neighbor bitmask and applying the resulting StairType.

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not create an implementation of this interface that stores per-instance state. The ruleset must be globally applicable and reusable.
- **Manual Invocation:** Avoid calling getStairType directly in game logic. Defer all connected block calculations to the engine's dedicated systems to ensure correctness and prevent desynchronization between the calculated state and the world state.

## Data Pipeline
This interface is a processing node in the block update data pipeline. It transforms raw neighbor data into a final, descriptive block shape.

> Flow:
> World Event (e.g., Block Placement) -> Neighbor Block State Query -> Neighbor Bitmask Generation -> **StairLikeConnectedBlockRuleSet.getStairType(bitmask)** -> New BlockState ID -> World Chunk Update -> Network Packet to Client

