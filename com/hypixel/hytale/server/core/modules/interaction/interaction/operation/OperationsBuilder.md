---
description: Architectural reference for OperationsBuilder
---

# OperationsBuilder

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.operation
**Type:** Transient

## Definition
```java
// Signature
public class OperationsBuilder {
```

## Architecture & Concepts

The OperationsBuilder is a foundational component of the server-side Interaction System. It functions as a high-level assembler for creating complex, stateful interaction sequences. These sequences, represented as an array of Operation objects, effectively form a low-level "script" or state machine that dictates an entity's behavior during an interaction, such as casting a spell, executing a combo attack, or using a tool.

This class provides a domain-specific language (DSL) for constructing these scripts. It abstracts the direct manipulation of an array into a structured, sequential build process. The core architectural concepts are:

*   **Operations:** Individual, atomic steps within an interaction (e.g., play sound, apply damage, wait for animation).
*   **Labels and Jumps:** A control-flow mechanism analogous to assembly language. Labels mark specific points within the operation sequence, and Jump operations allow the execution to branch, enabling loops and conditional logic within an interaction script.
*   **Immutability of Result:** The builder itself is a mutable, transient object. Its sole purpose is to produce an immutable `Operation[]` array via the `build` method. This resulting array is then cached and reused by the Interaction System, ensuring predictable and performant execution at runtime.

The internal `LabelOperation` class utilizes the Decorator pattern to attach metadata (the labels) to an operation without altering the operation's core logic. This metadata is then consumed by the execution context at runtime.

### Lifecycle & Ownership

The lifecycle of an OperationsBuilder is intentionally brief and confined to the configuration or setup phase of the server.

*   **Creation:** An OperationsBuilder is instantiated on-demand by systems responsible for defining game logic. This typically occurs when the server is loading and parsing asset definitions for items, skills, or NPC behaviors.
*   **Scope:** The instance exists only for the duration required to construct a single, complete `Operation[]` sequence. It is a short-lived, single-use object.
*   **Destruction:** Once the `build` method is called and the resulting array is retrieved, the OperationsBuilder instance has fulfilled its purpose. It should be discarded and is subsequently reclaimed by the Java garbage collector. It holds no persistent state and is not registered in any service context.

## Internal State & Concurrency

*   **State:** The internal state of the OperationsBuilder, primarily the `operationList`, is highly mutable. Each call to `addOperation` or `jump` modifies this internal list. This state is essential for the step-by-step construction of the final operation array.

*   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for synchronous, single-threaded construction of an interaction script. Concurrent access will lead to a corrupted operation list and unpredictable server behavior.

    **WARNING:** Confine each OperationsBuilder instance to the thread that created it. Do not store instances in shared fields or pass them between threads.

## API Surface

The public API is designed for a fluent, sequential build process.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| createLabel() | Label | O(1) | Creates a new label, immediately resolving it to the current position in the operation list. |
| createUnresolvedLabel() | Label | O(1) | Creates a placeholder label for forward-declarations. Must be resolved later. |
| resolveLabel(Label) | void | O(1) | Resolves a previously created unresolved label to the current position. Throws if already resolved. |
| jump(Label) | void | O(1) | Adds a `JumpOperation` to the sequence, creating a non-linear execution path. |
| addOperation(Operation) | void | O(1) | Appends a new operation to the end of the sequence. |
| build() | Operation[] | O(N) | Finalizes construction and returns the immutable array of operations. The builder should be discarded after this call. |

## Integration Patterns

### Standard Usage

The builder is used during a definition phase to construct an `Operation[]` array, which is then stored for later execution by the Interaction System.

```java
// Example: Building a simple "charge and fire" skill
OperationsBuilder builder = new OperationsBuilder();

// Define labels for control flow
Label chargeLoop = builder.createLabel();
Label fire = builder.createUnresolvedLabel();

// Sequence of operations
builder.addOperation(new PlaySoundOperation("charge_sound"));
builder.addOperation(new WaitOperation(1.5f), chargeLoop); // Loop back to this point
builder.addOperation(new CheckChargeLevelOperation(fire)); // Jump to 'fire' if fully charged
builder.jump(chargeLoop); // Continue charging

// Resolve the forward-declared label
builder.resolveLabel(fire);
builder.addOperation(new SpawnProjectileOperation("fireball"));
builder.addOperation(new ApplyCooldownOperation("fireball_skill"));

// Finalize and store the result
Operation[] skillSequence = builder.build();
SkillRegistry.register("fireball", skillSequence);
```

### Anti-Patterns (Do NOT do this)

*   **Instance Reuse:** Do not continue to use an OperationsBuilder instance after calling the `build` method. This can lead to unintended operations being added to a new sequence. Always create a new builder for each distinct script.
*   **Unresolved Labels:** Failing to call `resolveLabel` for every `createUnresolvedLabel` will result in runtime exceptions when the Interaction System attempts to execute a jump to an invalid location.
*   **Shared Instances:** Never share a single builder instance across multiple threads or systems building different scripts concurrently. This will corrupt the internal state.

## Data Pipeline

The OperationsBuilder is a producer component used during the server's configuration phase. It does not participate in the per-tick runtime data flow.

> **Build-Time Flow:**
> Game Asset (e.g., skill.json) -> Asset Parser -> **OperationsBuilder** -> `Operation[]` -> Skill Registry Cache

At runtime, the game engine retrieves the pre-built `Operation[]` from the cache and executes it; the OperationsBuilder is no longer involved.

