---
description: Architectural reference for BiLongToDoubleFunction
---

# BiLongToDoubleFunction

**Package:** com.hypixel.hytale.function.function
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface BiLongToDoubleFunction {
   double apply(long var1, long var3);
}
```

## Architecture & Concepts
BiLongToDoubleFunction is a functional interface that defines a behavioral contract. It represents a function that accepts two primitive **long** arguments and produces a primitive **double** result.

As a specialized functional interface, its primary architectural role is to provide a high-performance, low-overhead mechanism for passing behavior as data. It is a critical component in systems where performance is paramount, such as mathematical calculations in the physics engine, data processing pipelines, or performance-critical game logic.

By operating on primitive types, it completely avoids the memory and performance costs associated with autoboxing and unboxing `Long` and `Double` wrapper objects that would be required by the generic `java.util.function.BiFunction<Long, Long, Double>`. This makes it the preferred choice for any hot-path code that frequently performs this specific type of two-argument transformation.

## Lifecycle & Ownership
An interface does not have a lifecycle itself; rather, its *implementations* do.

- **Creation:** Implementations are not created with the **new** keyword. They are provided at the call-site via lambda expressions or method references. This is a compile-time construct.
- **Scope:** The lifetime of a BiLongToDoubleFunction implementation is tied to the object that holds a reference to it. It can be a short-lived variable within a method scope or a long-lived member of a service, such as a configurable strategy object.
- **Destruction:** The implementation object is eligible for garbage collection when no more references to it exist, following standard Java memory management rules.

## Internal State & Concurrency
- **State:** The interface contract is stateless. However, an implementation (i.e., a lambda expression) can become stateful if it captures variables from its enclosing scope. This is known as a closure.

    **WARNING:** Stateful lambdas, especially those that capture and modify mutable variables, can introduce significant complexity and are a common source of bugs.

- **Thread Safety:** The interface itself is thread-safe. The thread safety of a specific implementation is entirely dependent on its internal logic.
    - If an implementation is stateless (e.g., a pure function like `(a, b) -> a / (double)b`), it is inherently thread-safe.
    - If an implementation is stateful and modifies shared data without proper synchronization, it is **not** thread-safe. Extreme caution is required when using stateful lambdas in a concurrent environment.

## API Surface
The public contract consists of a single abstract method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(long, long) | double | O(N) | Executes the function. Complexity is entirely dependent on the specific implementation provided by the developer. |

## Integration Patterns

### Standard Usage
This interface is intended to be used as a method parameter, allowing callers to inject custom logic into a system.

```java
// Example: A system that calculates a damage modifier based on armor and health.
// The core calculation logic is passed in as a BiLongToDoubleFunction.

public class CombatSystem {
    public double calculateFinalDamage(long currentArmor, long maxHealth, BiLongToDoubleFunction modifierLogic) {
        double baseDamage = 100.0;
        double modifier = modifierLogic.apply(currentArmor, maxHealth);
        return baseDamage * modifier;
    }
}

// Usage:
CombatSystem combat = new CombatSystem();
BiLongToDoubleFunction armorScaling = (armor, health) -> 1.0 - (armor / (double)(armor + 500));
double finalDamage = combat.calculateFinalDamage(250L, 1000L, armorScaling);
```

### Anti-Patterns (Do NOT do this)
- **Using Generic Equivalents:** Do not use `BiFunction<Long, Long, Double>` in performance-sensitive code. The resulting autoboxing and unboxing will create unnecessary garbage and CPU overhead. Always prefer this primitive specialization.
- **Complex Inline Lambdas:** Avoid writing large, multi-line, or complex business logic directly inside a lambda expression. This harms readability and testability. For complex operations, implement the logic in a private static method and provide it as a method reference.
- **Unsynchronized Stateful Closures:** Never use a lambda that captures and modifies shared mutable state from a concurrent context without explicit synchronization. This is a direct path to race conditions and unpredictable behavior.

## Data Pipeline
This component is a behavioral primitive, not a data staging element. It acts as a functional "verb" within a larger data flow, transforming inputs into an output.

> Control Flow:
> Calling System -> Provides **long**, **long** inputs -> **BiLongToDoubleFunction.apply()** executes -> Returns **double** result -> Calling System continues execution

