---
description: Architectural reference for InternalReferenceResolver
---

# InternalReferenceResolver

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient

## Definition
```java
// Signature
public class InternalReferenceResolver {
```

## Architecture & Concepts
The InternalReferenceResolver is a stateful, single-use component that serves as the central authority for resolving forward-declared named references during the NPC asset compilation pipeline. Its primary function is to convert symbolic, string-based names within an NPC behavior script into a dense, integer-indexed graph of instruction builders. This process is critical for both performance and correctness.

At its core, the resolver acts as a symbol table and a dependency graph validator. As an NPC configuration is parsed, any reference to a named instruction block (e.g., a specific attack pattern) is passed to the resolver. The resolver interns this name, assigning it a unique, stable integer ID. It then builds up a dependency graph where nodes are the instruction builders and edges represent calls from one builder to another.

The system is designed in two distinct phases:
1.  **Resolution Phase:** The configuration is parsed, and all named references are registered via `getOrCreateIndex`. The corresponding builder implementations are supplied via `addBuilder`. During this phase, the resolver maintains string-to-integer mappings.
2.  **Validation & Finalization Phase:** After parsing is complete, `validateInternalReferences` is called to traverse the constructed graph, ensuring all references are valid and, critically, that no circular dependencies exist. Finally, the `optimise` method is called to discard the string-based mapping tables, reducing the memory footprint of the finalized asset.

This component is essential for enabling complex, non-linear NPC behaviors while guaranteeing their structural integrity before they are loaded into the game server.

### Lifecycle & Ownership
- **Creation:** A new InternalReferenceResolver is instantiated at the beginning of a single NPC asset compilation task. It is created by the higher-level asset parsing logic.
- **Scope:** The instance is scoped exclusively to the compilation of one NPC asset. It accumulates state throughout the parsing process and must not be shared or reused across different asset compilations.
- **Destruction:** The object is eligible for garbage collection after the NPC asset has been fully compiled and validated. The invocation of the `optimise` method signals the end of its primary lifecycle, after which it holds only the final, integer-indexed list of builders.

## Internal State & Concurrency
- **State:** The InternalReferenceResolver is highly mutable during the resolution phase. Its internal lists and maps (`builders`, `indexMap`, `nameMap`) are continuously populated as an asset file is processed. The `optimise` method represents a significant state transition, permanently removing the mapping data structures to conserve memory.
- **Thread Safety:** This class is **not thread-safe** and is designed for single-threaded access. Its internal state is not protected by locks or other concurrency primitives. All operations against a single instance must be performed sequentially by the same thread. Concurrent modification would lead to a corrupted state, race conditions, and unpredictable behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getOrCreateIndex(String name) | int | O(1) | Returns the integer index for a given name, creating a new one if it does not exist. Registers the index as a dependency if recording is active. |
| addBuilder(int index, BuilderInstructionReference builder) | void | O(1) | Associates a concrete builder instance with a previously created index. Throws IllegalStateException if a builder for that index already exists. |
| validateInternalReferences(String configName, List<String> errors) | void | O(V+E) | Traverses the entire dependency graph to check for missing builders and cyclic dependencies. Populates the errors list with any issues found. |
| getBuilder(int index, Class<?> classType) | Builder<T> | O(1) | Retrieves a resolved builder by its integer index. Throws an exception if the requested type is not supported. |
| setRecordDependencies() | void | O(1) | Enables the tracking of all indices retrieved via `getOrCreateIndex`. |
| stopRecordingDependencies() | void | O(1) | Disables dependency tracking. |
| optimise() | void | O(1) | Discards internal mapping tables to reduce memory usage. This is a destructive, one-way operation. |

## Integration Patterns

### Standard Usage
The resolver is used as part of a multi-stage compilation process. The typical lifecycle involves populating the resolver during parsing, validating its contents, and then optimizing it before final use.

```java
// 1. Create a resolver for a single NPC asset compilation
InternalReferenceResolver resolver = new InternalReferenceResolver();

// 2. During parsing, register names and add builders
// This would typically happen in a loop while reading a config file
int attackPatternIndex = resolver.getOrCreateIndex("unique_attack_pattern");
int idleBehaviorIndex = resolver.getOrCreateIndex("idle_behavior");

// ... later, when the builder for that name is defined
resolver.addBuilder(attackPatternIndex, new AttackPatternBuilder(...));
resolver.addBuilder(idleBehaviorIndex, new IdleBehaviorBuilder(...));

// 3. After parsing is complete, validate the entire graph
List<String> errors = new ArrayList<>();
resolver.validateInternalReferences("npc_config_alpha.json", errors);
if (!errors.isEmpty()) {
    throw new IllegalStateException("Validation failed: " + errors);
}

// 4. Discard mapping data to save memory
resolver.optimise();

// 5. Use the resolved builders to construct the final runtime object
Builder<Instruction> finalBuilder = resolver.getBuilder(attackPatternIndex, Instruction.class);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not re-use an InternalReferenceResolver instance for compiling a second, unrelated NPC asset. Its internal state is specific to a single asset, and re-use will cause name collisions and incorrect dependency graphs.
- **Modification After Optimization:** Do not call `getOrCreateIndex` or `addBuilder` after `optimise` has been invoked. This will result in a NullPointerException, as the internal maps it relies on have been discarded.
- **Skipping Validation:** Failure to call `validateInternalReferences` can allow structurally invalid assets (e.g., with circular dependencies) to be created. This will lead to difficult-to-diagnose runtime errors, such as server hangs or stack overflows.

## Data Pipeline
The InternalReferenceResolver is a key transformation stage in the data pipeline that converts a declarative NPC configuration file into an executable, in-memory representation.

> Flow:
> NPC Config File -> Parser -> **InternalReferenceResolver** (Converts names to indices, builds dependency graph) -> Validator (Detects cycles & missing links) -> Finalized `Instruction` Graph

