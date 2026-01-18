---
description: Architectural reference for InstructionType
---

# InstructionType

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Enumeration

## Definition
```java
// Signature
public enum InstructionType implements Supplier<String> {
```

## Architecture & Concepts
The InstructionType enum defines a finite, type-safe set of categories for NPC behavior instructions. It serves as a foundational component within the NPC asset building system, replacing primitive string-based identifiers with robust, compile-time constants. This prevents common errors related to typos or invalid instruction categories during NPC asset parsing and compilation.

This class is an example of the *Smart Enum* pattern. Beyond defining the constants, it provides pre-computed, highly optimized subsets of instructions (MotionAllowedInstructions, StateChangeAllowedInstructions) using EnumSet. These subsets are used system-wide for validation and conditional logic, allowing other systems to quickly determine if a given instruction type is permitted within a specific behavioral context (e.g., "Can this instruction modify the NPC's physical motion?").

The implementation of the Supplier interface provides a standardized way to retrieve a human-readable description, primarily for logging, debugging, and tooling purposes.

## Lifecycle & Ownership
- **Creation:** Enum constants are instantiated by the Java Virtual Machine (JVM) during class loading. They are not created dynamically; their existence is guaranteed at compile time.
- **Scope:** As static final instances, all InstructionType constants exist for the entire application lifecycle. They are effectively global, immutable singletons.
- **Destruction:** The enum instances are garbage collected only when their defining ClassLoader is unloaded, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** The state of InstructionType is entirely immutable. The internal *description* field is final and set via the private constructor. The public static EnumSet fields are also final and populated once during static initialization.
- **Thread Safety:** This class is inherently thread-safe. All state is immutable, and access to the enum constants and their associated data requires no synchronization. It can be safely accessed and used from any thread without risk of race conditions or data corruption.

## API Surface
The primary API consists of the enum constants themselves and the static validation sets.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Default | InstructionType | O(1) | Represents a default or fallback behavior instruction. |
| Interaction | InstructionType | O(1) | Represents an instruction triggered by player or world interaction. |
| Death | InstructionType | O(1) | Represents an instruction executed upon the NPC's death. |
| Component | InstructionType | O(1) | Represents a modular, reusable block of behavior. |
| StateTransitions | InstructionType | O(1) | Represents an instruction that defines transitions between states. |
| Any | EnumSet | O(1) | A pre-computed set containing all possible instruction types. |
| MotionAllowedInstructions | EnumSet | O(1) | A pre-computed set of types allowed to influence NPC movement. |
| StateChangeAllowedInstructions | EnumSet | O(1) | A pre-computed set of types allowed to trigger a state machine change. |
| get() | String | O(1) | Returns the human-readable description of the enum constant. |

## Integration Patterns

### Standard Usage
The primary use case is for control flow within the NPC behavior system and for validating asset definitions. Developers should use the constants for comparisons and the static sets for membership checks.

```java
// Example: Validating an instruction during asset parsing
void validateInstruction(Instruction instruction) {
    InstructionType type = instruction.getType();

    if (instruction.hasMotionData() && !InstructionType.MotionAllowedInstructions.contains(type)) {
        throw new AssetParseException("Instruction of type " + type + " cannot define motion.");
    }

    // Example: Control flow in the behavior processor
    switch (type) {
        case Death:
            // Execute death-specific logic
            break;
        case Interaction:
            // Execute interaction-specific logic
            break;
        default:
            // ...
    }
}
```

### Anti-Patterns (Do NOT do this)
- **String Comparison:** Never rely on the string description for logical comparisons. The string is for display purposes only and may change. Always compare the enum constants directly.
  ```java
  // BAD
  if (instruction.getType().get().equals("the death instruction")) { ... }

  // GOOD
  if (instruction.getType() == InstructionType.Death) { ... }
  ```
- **Extensibility:** Enums in Java are a closed set. Do not attempt to add new instruction types at runtime. If new types are needed, they must be added directly to the InstructionType enum source code.

## Data Pipeline
InstructionType acts as a data classifier and validator within the NPC asset loading pipeline. It does not process data itself but provides the metadata necessary for other systems to process it correctly.

> Flow:
> NPC Asset File (e.g., JSON) -> Asset Deserializer -> **InstructionType** (Classification & Validation) -> Behavior Tree Builder -> Runtime NPC Controller

