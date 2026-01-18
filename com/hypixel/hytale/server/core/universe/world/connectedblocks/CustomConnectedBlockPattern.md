---
description: Architectural reference for CustomConnectedBlockPattern
---

# CustomConnectedBlockPattern

**Package:** com.hypixel.hytale.server.core.universe.world.connectedblocks
**Type:** Configuration Object

## Definition
```java
// Signature
public class CustomConnectedBlockPattern extends CustomTemplateConnectedBlockPattern {
```

## Architecture & Concepts
The CustomConnectedBlockPattern class is a data-driven rule engine component that defines a single, matchable pattern for the "Connected Blocks" system. This system is responsible for dynamically changing a block's appearance based on its neighbors, enabling features like fences connecting, walls forming corners, and pipes joining together.

Each instance of this class represents one specific condition, or "pattern," that the game world is tested against. For example, a pattern might define the rule: "If a solid block exists at relative position (0, 1, 0) and an air block exists at (0, -1, 0), then I should transform into block X with rotation Y."

These patterns are not hard-coded. They are defined in external asset files and deserialized into memory at runtime using the provided static CODEC. This allows game designers to create complex block connection behaviors without modifying engine code. A single block type, like a fence, will have a list of these patterns, which are evaluated in order until one successfully matches the block's current surroundings.

The class's core responsibility is to evaluate its internal rules against the world state. This involves complex spatial queries and rotational transformations. It can handle simple block type checks, as well as more advanced conditions like matching "face tags" on neighboring blocks, allowing for highly specific connection logic. The system also supports programmatic generation of pattern variants through mirroring and rotation, significantly reducing the amount of configuration required from designers.

## Lifecycle & Ownership
-   **Creation:** Instances are exclusively created by the Hytale asset loading system via the static CODEC field. They are deserialized from block asset configuration files when the server starts or when assets are reloaded. They are typically contained within a parent CustomTemplateConnectedBlockRuleSet object.
-   **Scope:** An instance's lifetime is tied to the asset registry. It is effectively a singleton within the context of its definition file and persists for the entire server session. The object is immutable after its creation.
-   **Destruction:** Instances are eligible for garbage collection when the asset registry is cleared, which typically occurs during server shutdown.

## Internal State & Concurrency
-   **State:** The object's state is defined by its fields (e.g., rulesToMatch, patternRotationDefinition) which are populated once during deserialization. The object is considered immutable after this initial construction phase. The getConnectedBlockTypeKey method is purely computational and does not mutate instance state.

-   **Thread Safety:** This class is conditionally thread-safe. While its internal state is immutable, the core getConnectedBlockTypeKey method accesses a shared, static Random instance. Although it seeds the generator before each use to ensure deterministic output for a given coordinate, simultaneous calls from multiple threads could cause contention.

    **WARNING:** The engine must ensure that world update logic involving connected blocks is executed in a single-threaded context per world region to avoid potential concurrency issues with the static random number generator.

## API Surface
The primary public contract is the evaluation method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getConnectedBlockTypeKey(...) | Optional | O(T * R) | Evaluates this pattern against the world at a specific coordinate. Returns a ConnectedBlockResult containing the new block type and rotation if all rules match. The complexity is a function of the number of allowed Transformations (T) and the number of Rules (R) defined in the asset. |

## Integration Patterns

### Standard Usage
This class is not intended for direct developer interaction. It is invoked by higher-level systems that manage block updates. The engine retrieves the appropriate rule set for a block and iterates through its patterns, calling the evaluation method on each.

```java
// Conceptual example of engine-level usage
void onBlockUpdate(World world, Vector3i coordinate) {
    BlockType type = world.getBlockType(coordinate);
    CustomTemplateConnectedBlockRuleSet ruleSet = type.getConnectedBlockRuleSet();

    if (ruleSet != null) {
        // The engine iterates through all patterns for the block
        for (CustomConnectedBlockPattern pattern : ruleSet.getPatterns()) {
            Optional<ConnectedBlockResult> result = pattern.getConnectedBlockTypeKey(
                "defaultShape", world, coordinate, ruleSet, ...
            );

            if (result.isPresent()) {
                // A pattern matched, apply the transformation
                world.setBlock(coordinate, result.get().blockTypeKey(), result.get().rotation());
                return; // Stop after the first match
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new CustomConnectedBlockPattern()`. The object is uninitialized and invalid without being populated by the asset loader via its CODEC. All patterns must be defined in asset files.
-   **State Modification:** Do not attempt to modify the internal state (e.g., the rulesToMatch array) after the object has been loaded. This would violate its immutability contract and lead to unpredictable behavior across the server.
-   **Ignoring Context:** Calling getConnectedBlockTypeKey without a fully loaded world state (e.g., missing adjacent chunks) will cause the pattern matching to fail incorrectly. The method must be called within a context where all required neighbor data is available.

## Data Pipeline
The class functions as a decision-making step in the block update pipeline. It consumes world state and produces a potential state change.

> Flow:
> World Event (Block Placement/Update) -> Connected Blocks System -> RuleSet Retrieval -> **CustomConnectedBlockPattern.getConnectedBlockTypeKey** -> World State Query (Neighboring Blocks) -> Pattern Match Evaluation -> Optional<ConnectedBlockResult> -> World State Mutation (Set New Block)

