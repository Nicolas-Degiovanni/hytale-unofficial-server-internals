---
description: Architectural reference for BuilderParameters
---

# BuilderParameters

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient State Object

## Definition
```java
// Signature
public class BuilderParameters {
```

## Architecture & Concepts

The BuilderParameters class is a foundational component in the server-side NPC asset compilation pipeline. It serves as a stateful container and context manager for the parameters defined within an NPC asset file. Its primary role is to parse, store, and provide a compilation scope for custom variables and expressions that define an NPC's behavior and properties.

This class is not a passive data structure; it is an active participant in the transformation of declarative JSON asset data into executable bytecode. It encapsulates three critical concerns:

1.  **Parameter Storage:** It parses and holds a map of all parameters defined in a `Parameters` block of an asset file. Each parameter can have a value, type hints, validation logic, and visibility settings.
2.  **Expression Scoping:** It maintains an `StdScope` instance, which acts as a symbol table. Before expression compilation, the defined parameters are injected into this scope, making them available as variables to the expression compiler.
3.  **Compilation Context:** It manages a `CompileContext` object, which orchestrates the compilation of string-based expressions into a list of `ExecutionContext.Instruction` (bytecode). This bridges the gap between the high-level asset definition and the low-level execution engine.

The design supports hierarchical scoping through a copy constructor, allowing child assets or components to inherit and extend the parameters of their parents, a common pattern in complex game asset systems.

## Lifecycle & Ownership

-   **Creation:** Instances are created by the NPC asset loading system when a `Parameters` block is encountered in a JSON file. The constructors are `protected`, restricting instantiation to the asset builder framework within the same package. A new instance is typically created for each asset file or for each nested scope that defines its own parameters.
-   **Scope:** The lifetime of a BuilderParameters object is strictly bound to a single asset compilation task. It is a short-lived object that exists only during the parsing and compilation phase.
-   **Destruction:** The object becomes eligible for garbage collection as soon as the asset it was processing has been fully compiled. The `disposeCompileContext` method is a key part of its lifecycle, explicitly nullifying the reference to the compilation context to release memory associated with intermediate compilation artifacts.

## Internal State & Concurrency

-   **State:** BuilderParameters is a highly mutable, stateful object. Its internal collections—`parameters`, `scope`, and `dependencies`—are populated and modified throughout the asset parsing process. This state is essential for the sequential, multi-stage process of asset compilation.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be used by a single thread for the duration of a single asset compilation. Its internal state, particularly the `CompileContext`, is not protected by any synchronization mechanisms. Concurrent access would result in a corrupted scope, race conditions during compilation, and unpredictable system behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readJSON(JsonObject, StateMappingHelper) | void | O(N) | Parses a JSON object, populating the internal parameter map. N is the number of parameters. Throws IllegalStateException on duplicate keys or malformed data. |
| addParametersToScope() | void | O(N) | Injects the parsed parameters into the internal expression scope, making them available for compilation. |
| compile(String) | ValueType | O(M) | Compiles a string expression into bytecode using the current scope. M is the complexity of the expression. Returns the resulting value type. |
| createScope() | StdScope | O(S) | Creates a deep copy of the current scope, for use in isolated or nested expression evaluation. S is the number of symbols in the scope. |
| validateNoDuplicateParameters(BuilderParameters) | void | O(N) | Verifies that parameters from another scope (typically a child) do not conflict with this one. Throws IllegalStateException on conflict. |
| disposeCompileContext() | void | O(1) | Releases the internal reference to the compilation context, freeing associated memory. This should be called after all compilation for the asset is complete. |

## Integration Patterns

### Standard Usage

The BuilderParameters object is orchestrated by a higher-level asset parser. The typical flow involves creating an instance, populating it from JSON, adding its parameters to the scope, and then using it as a context for compiling other parts of the asset.

```java
// Hypothetical asset loader
BuilderParameters params = new BuilderParameters(parentScope, "npc_goblin.json", "");
JsonObject paramsJson = sourceFile.get("Parameters").getAsJsonObject();

// 1. Populate from the asset file
params.readJSON(paramsJson, stateHelper);

// 2. Make parameters available to the compiler
params.addParametersToScope();

// 3. Compile an expression from another part of the asset
String behaviorExpression = "HP > Parameters.lowHealthThreshold";
params.compile(behaviorExpression);
List<ExecutionContext.Instruction> bytecode = params.getInstructions();

// 4. Clean up compilation resources
params.disposeCompileContext();
```

### Anti-Patterns (Do NOT do this)

-   **State Reuse:** Do not reuse a BuilderParameters instance to compile a second, unrelated asset. The object will retain the scope, parameters, and dependencies from the first asset, leading to name collisions and incorrect behavior. Always create a new instance for each distinct compilation unit.
-   **Concurrent Compilation:** Do not pass a BuilderParameters instance to multiple threads to perform parallel compilation. The internal `CompileContext` and `scope` are not thread-safe and will become corrupted.
-   **Late Scope Modification:** Do not call `addParametersToScope` after compilation has already begun. The expression compiler relies on a stable scope during its analysis; modifying it mid-compile can lead to undefined behavior.

## Data Pipeline

The BuilderParameters class is a central stage in the data transformation pipeline that converts a declarative JSON asset into executable game logic.

> Flow:
> JSON Asset File -> JSON Parser -> `JsonObject` -> **BuilderParameters.readJSON** -> Internal Scope & Parameter Map -> **BuilderParameters.compile()** -> `List<Instruction>` (Bytecode) -> NPC Runtime Behavior

