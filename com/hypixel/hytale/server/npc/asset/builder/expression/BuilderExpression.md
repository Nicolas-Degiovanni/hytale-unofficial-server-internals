---
description: Architectural reference for BuilderExpression
---

# BuilderExpression

**Package:** com.hypixel.hytale.server.npc.asset.builder.expression
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class BuilderExpression {
```

## Architecture & Concepts

The BuilderExpression class is the cornerstone of the dynamic value system for server-side NPC assets. It provides a polymorphic, unified interface for values that can be either static (a literal number, string, or boolean) or dynamic (an expression evaluated at runtime). This abstraction is critical for defining complex and state-dependent NPC behaviors directly within asset files.

Architecturally, this class implements a variation of the **Interpreter** or **Strategy** design pattern. The abstract BuilderExpression defines the contract for how a value is retrieved, while concrete subclasses provide the specific implementation:
*   **Static Expressions** (e.g., BuilderExpressionStaticNumber, BuilderExpressionStaticString) represent literal values. They are immutable, high-performance wrappers around primitive data. Their evaluation is instantaneous and context-independent.
*   **Dynamic Expressions** (BuilderExpressionDynamic) represent expressions that must be evaluated at runtime. These objects parse and hold a representation of a script or expression string. Their evaluation depends on an external ExecutionContext, which supplies the necessary state (e.g., the NPC's current health, target distance, or world time).

The primary entry point into this system is through the static factory methods, particularly `fromJSON`. These factories act as the deserialization and parsing layer, consuming a JsonElement and dispatching to the correct concrete subclass. This design decouples the asset loading system from the internal implementation of any specific expression type.

## Lifecycle & Ownership
- **Creation:** BuilderExpression instances are created exclusively by the asset loading pipeline via the static `fromJSON` or `fromOperand` factory methods. They are instantiated when the server parses NPC definition files (typically JSON) during startup or a data reload. Direct instantiation of subclasses is a critical anti-pattern.

- **Scope:** An instance of a BuilderExpression persists for the entire lifetime of the in-memory NPC asset template it belongs to. These objects are considered part of the static game data definition and are shared across all NPC instances created from that template.

- **Destruction:** Instances are eligible for garbage collection only when the server unloads the associated NPC asset definitions, such as during a server shutdown or a full asset refresh.

## Internal State & Concurrency
- **State:** BuilderExpression objects are designed to be **immutable** after their initial creation during the asset parsing phase. Static variants hold a final, constant value. Dynamic variants hold a final, parsed representation of the expression logic. The `compile` method may be used to populate a cache or pre-calculate parts of the expression, but this is a one-time operation at load time.

- **Thread Safety:** This class is **thread-safe**. The evaluation methods (e.g., getNumber, getString) are stateless and do not modify the object's internal state. All runtime context required for evaluation is passed explicitly via the ExecutionContext parameter. This design allows a single BuilderExpression instance (part of an NPC template) to be safely and concurrently evaluated by multiple game threads, each representing a different NPC instance with its own unique ExecutionContext.

    **WARNING:** While the BuilderExpression itself is thread-safe, the provided ExecutionContext is not. Callers are responsible for ensuring that the ExecutionContext is not mutated by other threads during an evaluation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fromJSON(json, params) | BuilderExpression | O(N) | **Primary Factory.** Deserializes a JsonElement into a concrete BuilderExpression. Complexity is relative to the size of the JSON input. |
| fromOperand(operand) | BuilderExpression | O(1) | Factory for converting an internal ExecutionContext.Operand into a static BuilderExpression. |
| getType() | ValueType | O(1) | Returns the underlying data type (e.g., NUMBER, STRING) this expression will produce. |
| isStatic() | boolean | O(1) | Returns true if the expression represents a constant value that can be known without an ExecutionContext. |
| getNumber(context) | double | O(N) | Evaluates the expression and returns the result as a double. Throws IllegalStateException if the type is not a number. Complexity is O(1) for static types, O(N) for dynamic. |
| getString(context) | String | O(N) | Evaluates the expression and returns the result as a String. Throws IllegalStateException if the type is not a string. |
| getBoolean(context) | boolean | O(N) | Evaluates the expression and returns the result as a boolean. Throws IllegalStateException if the type is not a boolean. |
| compile(params) | void | O(N) | A load-time hook for pre-processing or validating a dynamic expression. Does nothing for static types. |

## Integration Patterns

### Standard Usage
The intended use is for a higher-level system, like an NPC asset loader, to create BuilderExpression objects during data parsing. These objects are then stored within an NPC template. At runtime, the game logic retrieves the expression and evaluates it with a context specific to a live NPC.

```java
// During asset loading
JsonElement healthJson = npcDefinition.get("maxHealth");
BuilderExpression healthExpression = BuilderExpression.fromJSON(healthJson, builderParameters);
npcTemplate.setMaxHealthExpression(healthExpression);

// During runtime for a specific NPC instance
ExecutionContext npcContext = createExecutionContextFor(liveNpc);
double currentMaxHealth = npcTemplate.getMaxHealthExpression().getNumber(npcContext);
liveNpc.setHealth(currentMaxHealth);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new BuilderExpressionStaticNumber(100)`. The static factory methods `fromJSON` and `fromOperand` are the sole entry points and contain critical dispatching logic. Bypassing them can lead to improperly configured objects.

- **Type Mismatch Evaluation:** Do not call an evaluation method that does not match the expression's type. Calling `getString()` on an expression for which `getType()` returns NUMBER will result in an `IllegalStateException`. Always check the type if it is not known ahead of time.

- **Ignoring `isStatic`:** For performance-critical code, check `isStatic()` at load time. If true, you can evaluate the expression once and cache the result, avoiding the overhead of passing an ExecutionContext repeatedly for a value that will never change.

## Data Pipeline
The BuilderExpression acts as the transformation layer between raw asset data and usable runtime game values.

> Flow:
> JSON Asset File -> Gson Deserializer -> JsonElement -> **BuilderExpression.fromJSON()** -> Concrete BuilderExpression Instance (Stored in Template) -> Runtime Evaluation with ExecutionContext -> Primitive Value (double, String, etc.) -> Game System (e.g., NPC Physics, AI)

