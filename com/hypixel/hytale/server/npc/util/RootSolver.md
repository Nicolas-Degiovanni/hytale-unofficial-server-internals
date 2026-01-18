---
description: Architectural reference for RootSolver
---

# RootSolver

**Package:** com.hypixel.hytale.server.npc.util
**Type:** Utility

## Definition
```java
// Signature
public class RootSolver {
```

## Architecture & Concepts
The RootSolver is a low-level, stateless mathematical utility class designed for finding the real roots of polynomial equations up to the fourth degree. It provides pure, static functions that implement analytical solutions for quadric, cubic, and quartic equations.

This component is foundational for higher-level systems that require complex geometric calculations. Its primary consumers are within the server's physics and AI engines, particularly for tasks involving:
- **Trajectory Prediction:** Calculating the time of impact between a projectile following a parabolic (quadric) or more complex path and a surface.
- **Intersection Testing:** Determining intersection points between complex curves used for NPC patrol paths, ability areas of effect, or procedural geometry.
- **Animation Kinematics:** Solving for joint angles or timing in complex, physics-driven animations.

RootSolver is intentionally isolated from the core game state and entity systems. It operates exclusively on numerical inputs and has no dependencies on the game world, making it a highly reusable and performant computational kernel.

## Lifecycle & Ownership
- **Creation:** The RootSolver class is never instantiated. As a class containing only static members, its methods are accessed directly on the class itself.
- **Scope:** The class and its methods are available for the entire lifetime of the server process, once loaded by the Java Virtual Machine.
- **Destruction:** The class is unloaded from memory when the server application terminates. There is no instance-level lifecycle to manage.

## Internal State & Concurrency
- **State:** RootSolver is completely **stateless**. It contains no instance fields and its only class-level fields are immutable static constants (M_PI, EQN_EPS). Each method call operates exclusively on the arguments provided to it.

- **Thread Safety:** This class is unconditionally **thread-safe**. Its methods are pure functions, meaning their output is determined solely by their input arguments, with no side effects or reliance on shared mutable state. It can be safely called from any number of concurrent threads, such as parallel AI behavior updates or physics calculations in a multi-threaded world simulation.

## API Surface
The public API consists of high-precision solvers. Note that all methods use an output parameter array for results to avoid performance penalties from heap allocations during frequent calls.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| solveQuadric(c2, c1, c0, results, resultIndex) | int | O(1) | Solves a quadric equation. Writes up to 2 roots into the results array starting at resultIndex. Returns the number of real roots found. |
| solveCubic(c3, c2, c1, c0, results) | int | O(1) | Solves a cubic equation. Writes up to 3 roots into the results array. Returns the number of real roots found. |
| solveQuartic(c4, c3, c2, c1, c0, results) | int | O(1) | Solves a quartic equation. Writes up to 4 roots into the results array. Returns the number of real roots found. |

**WARNING:** The caller is responsible for ensuring the provided `results` array is large enough to hold the maximum number of possible roots for the given equation. Failure to do so will result in an unhandled ArrayIndexOutOfBoundsException.

## Integration Patterns

### Standard Usage
Methods should be invoked statically. The caller must pre-allocate an array to store the roots and check the integer return value to determine how many valid roots were found.

```java
// Example: Find intersection times for a trajectory
double[] roots = new double[3]; // Must be large enough for a cubic equation
int numRoots = RootSolver.solveCubic(c3, c2, c1, c0, roots);

if (numRoots > 0) {
    // Process the first valid root, e.g., the smallest positive time
    double firstIntersectionTime = roots[0];
    // ...
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance with `new RootSolver()`. This provides no benefit, wastes memory, and signals a misunderstanding of the class design. All methods are static.
- **Ignoring Return Value:** The integer returned by each solve method indicates the number of valid roots written to the output array. Code that assumes a fixed number of roots or reads beyond the count of valid roots will process garbage data, leading to severe and difficult-to-trace bugs in AI or physics behavior.
- **Undersized Result Arrays:** Passing an array smaller than the maximum possible number of roots is a critical error. For `solveQuartic`, the `results` array must have a length of at least 4.

## Data Pipeline
As a computational utility, RootSolver does not participate in a traditional data pipeline. Instead, it functions as a single, synchronous processing stage within a larger algorithm.

> **Computational Flow:**
>
> High-Level System (e.g., Physics Engine) -> **RootSolver.solve(...)** -> High-Level System
> 1. The calling system (e.g., an AI behavior tree node) derives polynomial coefficients that model a physical or geometric problem.
> 2. The appropriate static solve method is invoked with the coefficients and a pre-allocated output array.
> 3. The RootSolver performs the calculation synchronously, populating the array with real roots.
> 4. The calling system uses the returned root count and the populated array to make a decision (e.g., adjust NPC path, register a collision).

