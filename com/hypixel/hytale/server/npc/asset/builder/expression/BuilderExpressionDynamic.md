---
description: Architectural reference for BuilderExpressionDynamic
---

# BuilderExpressionDynamic

**Package:** com.hypixel.hytale.server.npc.asset.builder.expression
**Type:** Transient

## Definition
```java
// Signature
public abstract class BuilderExpressionDynamic extends BuilderExpression {
```

## Architecture & Concepts
BuilderExpressionDynamic is an abstract base class representing a dynamically computed value within the NPC asset system. It serves as the runtime component for Hytale's server-side expression language, bridging declarative JSON asset configuration with the server's imperative execution logic.

The core architectural concept is **ahead-of-time compilation**. During server startup or asset loading, string-based expressions found in JSON files (e.g., `"player.distance < 10"`) are parsed and compiled into a more efficient, bytecode-like representation: an array of ExecutionContext.Instruction objects. An instance of a BuilderExpressionDynamic subclass then stores this pre-compiled instruction sequence.

This design avoids the significant performance overhead of parsing and interpreting the expression string each time it needs to be evaluated, which could be as frequent as every game tick for many NPCs. Instead, the game logic invokes the lightweight `execute` method, which processes the pre-compiled instructions against a given state.

This class and its concrete implementations (BuilderExpressionDynamicNumber, BuilderExpressionDynamicString, etc.) are central to creating reactive and data-driven NPC behaviors without requiring hardcoded logic.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the static factory method `fromJSON`. This factory is invoked by the higher-level asset loading system when it encounters a JSON object containing a "Compute" key. The factory uses a provided BuilderParameters context to compile the expression string and then instantiates the appropriate concrete subclass based on the expression's determined return type.
- **Scope:** The object's lifetime is bound to its containing NPC asset definition. It is an immutable part of the in-memory asset graph and persists as long as the asset is loaded by the server.
- **Destruction:** The object is marked for garbage collection when the server unloads the parent NPC asset definition. There is no manual destruction logic.

## Internal State & Concurrency
- **State:** The internal state, consisting of the original expression string and the compiled instructionSequence array, is **immutable** after construction. The object itself is a stateless executor.
- **Thread Safety:** This class is **thread-safe**. Its immutable nature allows a single instance to be safely referenced and executed by multiple threads simultaneously.

    **WARNING:** While the BuilderExpressionDynamic object is thread-safe, the ExecutionContext passed to the `execute` method is **not**. It is stateful and must not be shared across threads or concurrent executions. Each execution requires a unique or properly pooled ExecutionContext instance.

## API Surface
The primary interaction points are the static factory for creation and the protected `execute` method for evaluation by subclasses.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(ExecutionContext) | protected void | O(N) | Executes the pre-compiled instruction sequence against the provided context. N is the number of instructions. Throws IllegalStateException if the final result type on the context stack does not match the expected type. |
| fromJSON(JsonElement, BuilderParameters) | public static BuilderExpression | O(C) | The primary factory. Parses a JSON object, compiles the expression, and returns a new BuilderExpression instance. C is the complexity of expression compilation. |
| isStatic() | public boolean | O(1) | Returns false, indicating that the value is computed at runtime. This distinguishes it from a static BuilderExpression. |
| getExpression() | public String | O(1) | Returns the original, un-compiled expression string for debugging and logging. |
| toSchema() | public static Schema | O(1) | Generates a configuration schema for asset validation, defining the structure of a dynamic expression object. |

## Integration Patterns

### Standard Usage
Developers do not typically interact with this class directly. The engine's asset loader uses the `fromJSON` factory during asset parsing. Game systems then evaluate the expression via a higher-level abstraction, which internally calls the `execute` method.

```java
// High-level system code (e.g., in an AI behavior tree node)

// 1. The expression is loaded from JSON into an asset at startup
//    (This happens automatically via BuilderExpressionDynamic.fromJSON)
NPCBehaviorAsset asset = AssetManager.get("my_npc_behavior");
BuilderExpression condition = asset.getPrecondition();

// 2. During the game loop, the expression is evaluated with a fresh context
ExecutionContext context = new ExecutionContext();
context.setTarget(self);
context.setSource(player);

// 3. The execute method is called internally by the expression's own evaluation logic
boolean result = condition.evaluateAsBoolean(context);

if (result) {
    // ... perform action
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new BuilderExpressionDynamicNumber(...)`. The internal instruction sequence is complex and must be generated by the compilation pipeline accessed via the static `fromJSON` factory.
- **Context Reuse:** Do not share an ExecutionContext instance across concurrent evaluations. This will lead to race conditions and unpredictable behavior, as the context's internal stack and state will be corrupted.

## Data Pipeline
The flow of data from configuration to runtime evaluation is a multi-stage process orchestrated by the asset system.

> Flow:
> NPC Asset JSON File -> Asset Loader -> **BuilderExpressionDynamic.fromJSON** (compiles string to instructions) -> In-Memory Asset Graph -> Game Logic provides ExecutionContext -> **BuilderExpressionDynamic.execute** -> Result on ExecutionContext Stack -> Consumed by NPC Behavior System

