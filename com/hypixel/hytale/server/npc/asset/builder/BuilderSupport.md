---
description: Architectural reference for BuilderSupport
---

# BuilderSupport

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient

## Definition
```java
// Signature
public class BuilderSupport {
```

## Architecture & Concepts

The BuilderSupport class is a transient, stateful context object that serves as the central hub for compiling an NPC's behavior from a high-level asset definition into low-level, performance-optimized runtime data structures. It is the primary intermediary between a declarative NPC definition (parsed by a Builder) and the concrete data arrays that will be attached to an NPCEntity instance.

Its core architectural function is to perform **resource mapping and allocation**. During the asset build process, named resources like flags, timers, targets, and behavioral instructions are registered with BuilderSupport. The class translates these human-readable string identifiers into zero-based integer indices, or "slots". This translation is a critical optimization, allowing the game's performance-sensitive AI loop to operate on dense arrays with O(1) index-based access, rather than performing expensive string-based lookups.

BuilderSupport also manages the construction of hierarchical behavior components. Through a stack-based context system (see setToNewComponent and popComponent), it can scope resources and state machines to specific parts of an NPC's behavior tree, enabling the creation of complex, modular AI.

In essence, BuilderSupport acts as a compiler's symbol table and resource manager for the NPC AI system. It accumulates all necessary configuration, validates it, and finally provides the finalized, contiguous data arrays required to instantiate a fully-functional NPC.

## Lifecycle & Ownership

-   **Creation:** A BuilderSupport instance is created by the BuilderManager or a top-level Builder (e.g., RoleBuilder) at the very beginning of an NPC asset's build pipeline. It is provided with the target NPCEntity and other essential contexts like the EntityStore and ExecutionContext.

-   **Scope:** The object's lifetime is strictly scoped to the build process of a *single* NPC definition. It is a short-lived object that accumulates state throughout the parsing of one asset file.

-   **Destruction:** Once the top-level Builder completes its work, it extracts the final data arrays (e.g., via allocateFlags, getInstructionSlotMappings) from the BuilderSupport instance. After this point, the BuilderSupport object holds no further responsibilities and is discarded, becoming eligible for garbage collection. It holds no state that persists beyond the build phase.

## Internal State & Concurrency

-   **State:** BuilderSupport is highly mutable by design. Its primary purpose is to aggregate state from various parts of the build process. It contains numerous maps, lists, and builders that are populated as an NPC asset is parsed. Key state includes slot mappings for various resource types, a growing list of instructions, and configuration flags for required subsystems.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for synchronous, single-threaded execution within the scope of a single NPC asset build. Concurrent access would result in corrupted slot mappings, race conditions when allocating new slots, and an indeterminate final state, leading to severe and difficult-to-diagnose runtime bugs in the resulting NPC.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getFlagSlot(String name) | int | O(1) | Maps a named flag to an integer slot, creating one if it does not exist. |
| getInstructionSlot(String name) | int | O(1) | Allocates a slot for a named instruction, returning its index. |
| putInstruction(int slot, Instruction instruction) | void | O(1) | Places a fully constructed Instruction into its pre-allocated slot. Throws if the slot is occupied. |
| setToNewComponent() | void | O(1) | Pushes the current component context onto a stack and begins a new one. Essential for building nested behaviors. |
| popComponent() | void | O(1) | Pops the component context from the stack, returning to the parent context. |
| allocateFlags() | boolean[] | O(N) | Creates and returns a boolean array sized to fit all registered flags. Returns null if no flags were registered. |
| getInstructionSlotMappings() | Instruction[] | O(N) | Validates and returns the final array of all registered Instruction objects. Throws if any slot is unassigned. |

## Integration Patterns

### Standard Usage

BuilderSupport is not used directly by game logic developers. It is an internal tool for authors of new NPC Builders. A Builder implementation uses it to translate its parsed asset data into engine-compatible structures.

```java
// Within a hypothetical Builder's parse method
void parseBehavior(BuilderSupport support, BehaviorAsset asset) {
    // Register a named flag. Support returns slot 0.
    int isAlertSlot = support.getFlagSlot("isAlert");

    // Allocate a slot for a named instruction. Support returns slot 0.
    int patrolInstructionSlot = support.getInstructionSlot("patrol_routine");

    // Create the instruction logic
    Instruction patrolInstruction = new PatrolInstruction(isAlertSlot);

    // Place the fully-formed instruction into its allocated slot
    support.putInstruction(patrolInstructionSlot, patrolInstruction);
}
```

### Anti-Patterns (Do NOT do this)

-   **State Reuse:** Do not attempt to reuse a BuilderSupport instance to build a second NPC. Its internal state is specific to a single build process and must be discarded. Always create a new instance for each NPC asset.
-   **Direct Instantiation:** Game logic should never instantiate BuilderSupport directly. It is managed and provided by the NPC asset building framework.
-   **Concurrent Modification:** Do not access or modify a BuilderSupport instance from multiple threads. The build process for a single asset must be fully serialized.
-   **Late Registration:** Do not attempt to register new slots (e.g., getFlagSlot) after the final allocation methods (e.g., allocateFlags) have been called. The size of the final arrays is fixed upon allocation.

## Data Pipeline

BuilderSupport functions exclusively within a build-time data transformation pipeline. It does not exist at runtime.

> Flow:
> NPC Asset File (JSON/XML) -> `Builder` Parser -> **BuilderSupport (Resource Mapping & State Aggregation)** -> Finalized Runtime Arrays (`Instruction[]`, `boolean[]`, `Vector3d[]`, etc.) -> Attachment to `NPCEntity` instance

