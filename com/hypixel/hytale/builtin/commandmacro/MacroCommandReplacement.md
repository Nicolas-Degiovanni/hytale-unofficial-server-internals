---
description: Architectural reference for MacroCommandReplacement
---

# MacroCommandReplacement

**Package:** com.hypixel.hytale.builtin.commandmacro
**Type:** Value Object

## Definition
```java
// Signature
public class MacroCommandReplacement {
```

## Architecture & Concepts
The MacroCommandReplacement class is a fundamental, immutable data structure within the command macro system. It serves as a blueprint for a single substitution operation, encapsulating the relationship between a placeholder in a macro template and its final, resolved value.

This class is not a service or a manager; it is a pure data container. Its primary role is to decouple the macro definition (the *what*) from the macro execution engine (the *how*). A collection of MacroCommandReplacement objects collectively defines the complete transformation for a given macro, allowing the processing engine to be stateless and generic.

For example, in a command like `/teleport $target_player --silent`, two MacroCommandReplacement instances would be created: one to handle the substitution for *$target_player* and another to handle the optional argument *--silent*.

### Lifecycle & Ownership
- **Creation:** Instances are created by the macro parsing and configuration layer whenever a macro with replaceable arguments is defined or loaded. This is a direct instantiation via its constructor.
- **Scope:** The object's lifetime is tied to the lifecycle of the macro definition it belongs to. It is typically a short-lived, transient object used during the command expansion phase.
- **Destruction:** As a simple value object with no external resource handles, it is managed entirely by the Java garbage collector. It is eligible for cleanup as soon as the macro expansion process completes and no further references exist.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields are declared final and are assigned exclusively within the constructor. Once an instance is created, its state cannot be altered.
- **Thread Safety:** **Inherently thread-safe**. Due to its immutable design, a MacroCommandReplacement instance can be safely read, shared, and processed by multiple threads concurrently without any risk of data corruption or race conditions. No external locking or synchronization is required.

## API Surface
The public contract is minimal, providing read-only access to the replacement parameters.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getNameOfReplacingArg() | String | O(1) | Returns the placeholder string that will be replaced (e.g., $target). |
| getOptionalArgumentKey() | @Nullable String | O(1) | Returns the formatted optional argument key (e.g., --key ) or null if not applicable. |
| getStringToReplaceWithValue() | String | O(1) | Returns the concrete value to be substituted into the command string. |

## Integration Patterns

### Standard Usage
This class is intended to be used as part of a collection to transform a template command string into an executable one. The macro processing system will typically iterate over a list of these objects.

```java
// A macro processor receives a list of replacements
List<MacroCommandReplacement> replacements = List.of(
    new MacroCommandReplacement("$player", "Herobrine"),
    new MacroCommandReplacement("$coords", "0 64 0", "location")
);

String template = "/tp $player --location $coords";
String result = template;

// The processor iterates and applies each replacement rule
for (MacroCommandReplacement rep : replacements) {
    result = result.replace(rep.getNameOfReplacingArg(), rep.getStringToReplaceWithValue());
}

// result is now "/tp Herobrine --location 0 64 0"
```

### Anti-Patterns (Do NOT do this)
- **State Mutation:** Do not attempt to modify an instance after creation. The class is immutable by design. If a different replacement rule is needed, create a new MacroCommandReplacement instance.
- **Manual Key Formatting:** Do not manually prepend dashes to the optional argument key when constructing the object. The constructor handles the formatting internally.
    - **BAD:** `new MacroCommandReplacement("val", "val", "--key")`
    - **GOOD:** `new MacroCommandReplacement("val", "val", "key")`

## Data Pipeline
MacroCommandReplacement acts as a configuration object that directs a data transformation pipeline, rather than being a stage within it. It holds the rules that the processor uses to operate on the data.

> Flow:
> Macro Configuration (File/UI) -> Parser -> **MacroCommandReplacement Instance(s)** -> Macro Processor -> Final Command String -> Command Bus

