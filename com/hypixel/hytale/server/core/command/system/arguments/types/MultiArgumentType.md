---
description: Architectural reference for MultiArgumentType
---

# MultiArgumentType

**Package:** com.hypixel.hytale.server.core.command.system.arguments.types
**Type:** Framework Component / Abstract Base Class

## Definition
```java
// Signature
public abstract class MultiArgumentType<DataType> extends ArgumentType<DataType> {
```

## Architecture & Concepts
MultiArgumentType is an abstract base class that forms the foundation of the server's composite command argument parsing system. It implements the **Composite Pattern**, allowing multiple, distinct `SingleArgumentType` instances to be grouped and treated as a single, cohesive argument. This enables the creation of complex, structured argument types from simpler primitives, such as a `Vec3` argument composed of three `Double` arguments.

The core responsibility of this class is to orchestrate the parsing process. It iterates through a predefined sequence of child argument types, feeding each one a corresponding slice of the raw input string array. If all child parsers succeed, it aggregates their results into a `MultiArgumentContext`.

Crucially, MultiArgumentType employs the **Template Method Pattern**. The final step of converting the populated `MultiArgumentContext` into the specific, high-level `DataType` is delegated to the concrete subclass via the abstract `parse(MultiArgumentContext, ParseResult)` method. This separates the generic parsing orchestration logic from the domain-specific object construction.

## Lifecycle & Ownership
- **Creation:** Concrete subclasses of MultiArgumentType are instantiated once during server bootstrap, typically within a central command or argument type registry. They are designed as definition objects or templates, not as per-command-execution instances.
- **Scope:** An instance of a MultiArgumentType subclass is a stateless singleton within the argument parsing system. It persists for the entire lifetime of the server.
- **Destruction:** The object is garbage collected during server shutdown as part of the application's final cleanup. No explicit destruction logic is required.

## Internal State & Concurrency
- **State:** The primary internal state is the `argumentValues` map, which holds the ordered collection of child argument types. This state is **immutable after initialization**. The `withParameter` method is exclusively intended for use within the constructor of a subclass. Any attempt to modify this collection after the initial setup phase is a severe design violation.
- **Thread Safety:** This class is **conditionally thread-safe**.
    - **Initialization Phase:** The construction and configuration via `withParameter` is **not thread-safe** and MUST be performed by a single thread during the server's initial loading sequence.
    - **Parsing Phase:** The `parse` methods are **fully thread-safe**. The internal state is read-only during parsing, and all other variables are confined to the method's stack frame. A single MultiArgumentType instance can be safely used to parse commands from multiple players or systems concurrently.

## API Surface
The public contract is focused on subclass implementation and the core parsing operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| withParameter(name, usage, argumentType) | WrappedArgumentType | O(1) | **Builder Method.** Registers a child argument type. Must only be called during construction. Throws IllegalArgumentException on duplicate names. |
| parse(input, parseResult) | DataType | O(N) | Orchestrates the parsing of child arguments. N is the number of registered child arguments. Returns null on failure. |
| parse(context, parseResult) | DataType | *Abstract* | **Template Method.** Implemented by subclasses to construct the final data object from successfully parsed child arguments. |

## Integration Patterns

### Standard Usage
A concrete implementation defines its constituent parts in its constructor and implements the final object assembly logic in the abstract `parse` method.

```java
// Example: A hypothetical argument type for a 2D integer vector (e.g., "10 20")
public class Vec2iArgumentType extends MultiArgumentType<Vector2i> {

    public Vec2iArgumentType() {
        super("vec2i", "<x> <y>", "10 20", "-5 100");
        // Define the structure during construction
        this.withParameter("x", "<x>", new IntegerArgumentType());
        this.withParameter("y", "<y>", new IntegerArgumentType());
    }

    @Override
    @Nullable
    public Vector2i parse(@Nonnull MultiArgumentContext context, @Nonnull ParseResult parseResult) {
        // Assemble the final object from the context
        int x = context.get("x");
        int y = context.get("y");
        return new Vector2i(x, y);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Dynamic Reconfiguration:** Do not expose the `withParameter` method publicly or call it outside of the constructor. The argument structure is meant to be static and defined at startup.
- **Ignoring ParseResult:** The `parseResult` object must be checked for failure after each parsing operation. The framework's `parse` loop does this, but custom implementations must be equally diligent.
- **Stateful Subclasses:** Subclasses must not contain mutable state. They are shared templates and storing per-execution data will lead to severe concurrency issues.

## Data Pipeline
MultiArgumentType acts as a dispatcher and aggregator in the command parsing pipeline. It deconstructs the input stream for its children and then reconstructs a final object from their results.

> Flow:
> Raw String Array -> Command Dispatcher -> **MultiArgumentType.parse(String[], ...)** -> Slices input for each child -> Child `SingleArgumentType.parse()` -> Results collected in **MultiArgumentContext** -> **Subclass.parse(MultiArgumentContext, ...)** -> Final `DataType` Object -> Command Executor

