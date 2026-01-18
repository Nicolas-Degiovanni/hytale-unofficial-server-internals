---
description: Architectural reference for BuilderHeadMotionBase
---

# BuilderHeadMotionBase

**Package:** com.hypixel.hytale.server.npc.corecomponents.builders
**Type:** Base Class / Template

## Definition
```java
// Signature
public abstract class BuilderHeadMotionBase extends BuilderMotionBase<HeadMotion> {
```

## Architecture & Concepts
The BuilderHeadMotionBase class is an abstract template within the server-side Non-Player Character (NPC) instruction framework. It serves as a specialized base for all builder classes that are responsible for constructing **HeadMotion** instructions.

Architecturally, this class enforces a design contract. By extending the generic BuilderMotionBase with the concrete type HeadMotion, it ensures that all its subclasses are exclusively associated with creating instructions that control an NPC's head. The primary mechanism for this association is the overridden `category` method, which hardcodes the return type to HeadMotion.class.

This design allows a central instruction factory or registry to dynamically identify and use the correct family of builders for a given instruction type, decoupling the instruction creation logic from the NPC's core behavior tree.

### Lifecycle & Ownership
- **Creation:** This class is abstract and is never instantiated directly. Concrete subclasses (e.g., BuilderHeadLookAtEntity) are instantiated on-demand by the NPC instruction system when a behavior script requires the creation of a head motion.
- **Scope:** Instances of its subclasses are **transient and short-lived**. A new builder is typically created for the construction of a single instruction and is discarded immediately after the `build` method is called.
- **Destruction:** The builder object is eligible for garbage collection as soon as the resulting HeadMotion instruction has been created and the builder reference goes out of scope.

## Internal State & Concurrency
- **State:** This base class is stateless. However, its concrete subclasses are inherently stateful during the building process, holding temporary configuration data such as a target entity or world coordinates. This state is mutable.
- **Thread Safety:** This class and its subclasses are **not thread-safe**. They are designed to be used within the context of a single NPC's update tick. Sharing a builder instance across multiple threads or reusing a configured builder will result in unpredictable behavior and severe race conditions.

## API Surface
The public contract of this specific class is minimal, as its primary purpose is to establish a type-safe hierarchy. The main API surface is defined in its parent classes and implemented in its children.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| category() | Class<HeadMotion> | O(1) | Returns the class literal for HeadMotion. This is a final method used by the framework to identify the instruction category this builder family produces. |

## Integration Patterns

### Standard Usage
A developer will never interact with BuilderHeadMotionBase directly. Instead, they will use a concrete implementation provided by a higher-level NPC behavior API. The pattern involves chaining configuration methods and finalizing with a build call.

```java
// Hypothetical example of using a concrete subclass
// Note: The builder is provided by the 'npc.instructions()' factory method.

HeadMotion lookAtPlayer = npc.instructions()
    .head() // Gets a factory for head-related builders
    .lookAtEntity(targetPlayer) // Returns a concrete builder and configures it
    .withDuration(5000) // Further configuration
    .build(); // Creates the final HeadMotion instruction

npc.getInstructionQueue().add(lookAtPlayer);
```

### Anti-Patterns (Do NOT do this)
- **Manual Instantiation:** Never instantiate a concrete subclass (e.g., `new BuilderHeadLookAtEntity()`). The NPC instruction system manages the lifecycle of these objects.
- **Builder Reuse:** Do not hold a reference to a builder after calling `build()` with the intent to reuse it. Each instruction requires a new, clean builder instance obtained from the framework.

## Data Pipeline
This class is not part of a data stream; it is a factory component in a construction pipeline. Its role is to transform configuration method calls into a structured data object.

> Flow:
> NPC Behavior Script -> Instruction Factory -> **Concrete Subclass of BuilderHeadMotionBase** -> `build()` Invocation -> Immutable `HeadMotion` Object -> NPC Instruction Queue

