---
description: Architectural reference for FlagOperator
---

# FlagOperator

**Package:** com.hypixel.hytale.server.worldgen.util.condition.flag
**Type:** Utility

## Definition
```java
// Signature
public enum FlagOperator implements IntBinaryOperator {
```

## Architecture & Concepts
The FlagOperator enum provides a type-safe, object-oriented representation of fundamental bitwise operations. It encapsulates the logic for AND, OR, and XOR operations on integer bitmasks, conforming to the Java functional interface IntBinaryOperator.

Architecturally, this enum employs the **Strategy Pattern**. Each enum constant (And, Or, Xor) is a concrete, stateless strategy for combining two integer values. This design allows bitwise logic to be passed as a first-class object, enabling more flexible and descriptive APIs within the world generation system.

Its primary role is to serve as a core utility for evaluating complex, flag-based conditions. For instance, a world generation rule might need to check if a block's state contains a combination of flags (e.g., *IS_WET* AND *HAS_MOSS*). By using FlagOperator, this logic becomes explicit and less prone to errors from incorrect operator usage.

### Lifecycle & Ownership
- **Creation:** Instances are created and initialized by the Java Virtual Machine (JVM) during class loading. As an enum, its constants are compile-time singletons.
- **Scope:** Application-wide. The And, Or, and Xor instances persist for the entire lifetime of the server process.
- **Destruction:** Instances are destroyed only when the JVM shuts down. There is no manual memory management or cleanup required.

## Internal State & Concurrency
- **State:** **Immutable and Stateless**. FlagOperator constants contain no fields and hold no data. They are pure functions that only encapsulate behavior.
- **Thread Safety:** **Inherently Thread-Safe**. Due to its stateless nature, a FlagOperator constant can be safely shared and invoked by any number of threads concurrently without requiring any synchronization mechanisms.

## API Surface
The public contract is defined by the IntBinaryOperator interface, making it interchangeable with any other function that operates on two integers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| applyAsInt(left, right) | int | O(1) | Executes the specific bitwise operation (AND, OR, XOR) on the two integer operands. This is the primary contract. |

## Integration Patterns

### Standard Usage
FlagOperator is intended to be used directly as a function object, often passed into a higher-order function or a system that evaluates conditions.

```java
// Example: Combining world state flags for a generation condition.
import com.hypixel.hytale.server.worldgen.util.condition.flag.FlagOperator;

// Assume these flags represent different terrain properties.
final int IS_MOUNTAIN = 0b0001; // 1
final int IS_FOREST   = 0b0010; // 2
final int IS_WET      = 0b0100; // 4

int currentFlags = IS_MOUNTAIN | IS_FOREST; // Current flags are 3 (0b0011)
int requiredFlags = IS_MOUNTAIN;

// Use the 'And' operator to check if the required flag is present.
int result = FlagOperator.And.applyAsInt(currentFlags, requiredFlags);

if (result == requiredFlags) {
    // This block will execute because (0b0011 & 0b0001) equals 0b0001.
    System.out.println("Condition met: The area is a mountain.");
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Enums cannot be instantiated using the *new* keyword. Attempting to do so will result in a compile-time error. Always reference the static constants directly (e.g., FlagOperator.And).
- **Logical Confusion:** Do not mistake these bitwise operators for their boolean logical counterparts (&&, ||). Using FlagOperator.Or to evaluate two non-zero integers will not produce a boolean-equivalent result and indicates a fundamental misunderstanding of the component's purpose. It is designed exclusively for bitmask manipulation.

## Data Pipeline
As a pure function, FlagOperator acts as a simple, stateless processing node in a data flow. It takes two integers as input and produces a single integer as output, with no side effects.

> Flow:
> Integer Bitmask A -> **FlagOperator** -> Resulting Integer Bitmask
> Integer Bitmask B -> **FlagOperator**

