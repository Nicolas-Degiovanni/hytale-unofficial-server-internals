---
description: Architectural reference for StdLib
---

# StdLib

**Package:** com.hypixel.hytale.server.npc.util.expression
**Type:** Singleton

## Definition
```java
// Signature
public class StdLib extends StdScope {
```

## Architecture & Concepts
The StdLib class is a foundational component of the server-side NPC expression language engine. It establishes a global, immutable, root-level scope containing a standard library of constants and functions. This class acts as the "built-in" provider of common utilities, such as mathematical operations, random number generation, and string or array checks.

Architecturally, it is designed as a stateless, read-only dictionary of functions. It is not an evaluator itself but rather a dependency consumed by an expression evaluator. By centralizing these common functions, the system ensures consistent behavior for all scripts and simplifies the creation of new expression contexts, which can inherit from this base scope.

The distinction between *invariant* and *variant* functions is a key design choice. Invariant functions (e.g., `max`, `min`) are pure and will always produce the same output for a given input. Variant functions (e.g., `random`) are impure and can produce different outputs on subsequent calls. This classification allows the expression engine to potentially perform optimizations, such as constant folding or result caching, for invariant sub-expressions.

## Lifecycle & Ownership
- **Creation:** The single instance is created statically and lazily by the JVM when the `getInstance` method is first invoked. The private constructor ensures that no other instances can be created.
- **Scope:** The instance is application-scoped. It persists for the entire lifetime of the server process once loaded.
- **Destruction:** The object is garbage collected only when the server process terminates and the class loader is unloaded. There is no explicit destruction or cleanup mechanism.

## Internal State & Concurrency
- **State:** The internal state of StdLib is populated once within its private constructor and is effectively **immutable** thereafter. It holds a collection of function definitions and constants that do not change during runtime.
- **Thread Safety:** This class is **thread-safe**. Its immutable nature means it can be safely shared across all threads without synchronization. Furthermore, the variant functions it provides, such as `random`, are implemented using thread-safe mechanisms like `ThreadLocalRandom` to prevent contention in a highly concurrent server environment.

## API Surface
The primary programmatic interaction is retrieving the singleton instance. The effective API consists of the functions it registers into the expression context.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getInstance() | StdScope | O(1) | Returns the global singleton instance of the standard library. |

### Registered Functions

| Name | Type | Notes |
| :--- | :--- | :--- |
| max | Invariant | Returns the greater of two numbers. |
| min | Invariant | Returns the lesser of two numbers. |
| isEmpty | Invariant | Checks if a string is null or empty. |
| isEmptyStringArray | Invariant | Checks if a string array has zero length. |
| isEmptyNumberArray | Invariant | Checks if a number array has zero length. |
| makeRange | Invariant | Creates a number array of two identical values. |
| random | Variant | Returns a random double between 0.0 and 1.0. |
| randomInRange | Variant | Returns a random double within a specified range. |

## Integration Patterns

### Standard Usage
StdLib is intended to be used as a foundational scope for an expression evaluation context. An evaluator would retrieve the singleton and use it as the root of its scope chain.

```java
// An ExpressionEvaluator would retrieve the singleton to build its context
StdScope rootScope = StdLib.getInstance();

// The evaluator would then execute expressions against a context
// that inherits from this root scope.
// (Conceptual example, evaluator not shown)
ExpressionEvaluator evaluator = new ExpressionEvaluator(rootScope);
boolean result = evaluator.evaluate("max(5, 10) > 8"); // true
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Do not attempt to create an instance of StdLib using reflection. This would violate the singleton pattern and is unsupported.
- **Runtime Modification:** Do not attempt to cast the returned StdScope and modify its contents. The standard library is designed to be a global, immutable constant. Modifying it at runtime will lead to unpredictable behavior across all server scripts.
- **Assuming Determinism:** Do not rely on variant functions like `random` to produce the same result twice. Scripts requiring deterministic outcomes must avoid these functions or provide their own seeded random number generator.

## Data Pipeline
StdLib does not process a flow of data itself. Instead, it serves as a functional dependency for systems that do. It injects functionality into an expression evaluation pipeline.

> Flow:
> NPC Behavior Script (Text) -> Expression Parser -> Abstract Syntax Tree (AST) -> **Expression Evaluator** (which queries **StdLib** for function definitions) -> Behavior Outcome (e.g., boolean, number)

