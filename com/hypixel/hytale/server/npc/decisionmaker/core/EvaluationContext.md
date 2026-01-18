---
description: Architectural reference for EvaluationContext
---

# EvaluationContext

**Package:** com.hypixel.hytale.server.npc.decisionmaker.core
**Type:** Transient

## Definition
```java
// Signature
public class EvaluationContext {
```

## Architecture & Concepts
The EvaluationContext is a parameter object that encapsulates the configuration for a single Non-Player Character (NPC) decision-making cycle. It is a core component of the Utility AI system, acting as a transient data container that dictates the constraints and modifiers for an evaluation.

Instead of passing numerous primitive parameters to the decision-making logic, this class bundles them into a single, coherent unit. This design pattern improves API stability and readability, allowing the core evaluation algorithm to remain decoupled from the specific parameters that influence it. The context defines *how* an evaluation should be performed, including thresholds for action viability (utility), the influence of various factors (weight), and the degree of randomness in the final choice (predictability).

It is fundamentally a state-carrier, designed to be created, configured, used, and then discarded within a very short timeframe.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand by the higher-level AI processing loop, typically within a DecisionMaker or a related service, immediately before an NPC needs to evaluate its next action. Given its lightweight nature, direct instantiation is common, though object pooling is a viable optimization strategy to reduce garbage collection overhead.

- **Scope:** The lifetime of an EvaluationContext is exceptionally brief, typically scoped to a single method call or, at most, a single server tick for one NPC's AI evaluation. It is not intended to persist between ticks or be stored as part of an NPC's long-term state.

- **Destruction:** The object becomes eligible for garbage collection as soon as the evaluation method it was passed to completes. If an object pool is used, the context is returned to the pool via its `reset` method for reuse.

## Internal State & Concurrency
- **State:** The EvaluationContext is entirely mutable. Its primary purpose is to hold a set of configurable parameters that are modified before being passed into the evaluation system. It does not cache any data and its state is self-contained.

- **Thread Safety:** **This class is not thread-safe.** It contains no synchronization primitives and is designed for use within a single thread, such as the main server thread or a dedicated AI worker thread for a specific entity. Sharing an instance across concurrent evaluations for different NPCs will result in race conditions and highly unpredictable AI behavior. Each evaluation must operate on a unique EvaluationContext instance.

## API Surface
The public API is composed of simple accessors and mutators for configuring an evaluation pass.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setMinimumUtility(double) | void | O(1) | Sets the minimum utility score an option must have to be considered. Throws IllegalArgumentException if the value is not between 0.0 and 1.0. |
| setMinimumWeightCoefficient(double) | void | O(1) | Sets the minimum weight a consideration must have to influence the score. Throws IllegalArgumentException if the value is negative. |
| setPredictability(float) | void | O(1) | Configures the determinism of the choice. A value of 1.0 is fully predictable; 0.0 is highly random. Throws IllegalArgumentException if the value is not between 0.0 and 1.0. |
| setLastUsedNanos(long) | void | O(1) | Records a timestamp, typically used to implement cooldowns on recently performed actions. |
| reset() | void | O(1) | Resets all parameters to their default state, preparing the object for reuse in a new evaluation. |

## Integration Patterns

### Standard Usage
The correct pattern involves acquiring a context, resetting it to a known-default state, configuring it for the specific evaluation, and then passing it to the decision-making system.

```java
// Assumes an Evaluator service and a specific NPC
Evaluator npcEvaluator = server.getAIService().getEvaluator();
EvaluationContext context = new EvaluationContext(); // Or acquire from a pool

// Configure context for this specific evaluation
context.reset();
context.setPredictability(0.9f); // Make this NPC's behavior highly predictable
context.setMinimumUtility(0.2); // Ignore actions with very low scores

// Pass the configured context to the core logic
Decision bestDecision = npcEvaluator.selectBestDecision(npc, availableOptions, context);

// The context can now be discarded
```

### Anti-Patterns (Do NOT do this)
- **Reusing Without Reset:** Reusing an EvaluationContext instance for a new evaluation without first calling `reset` is a critical error. This will cause parameters from the previous decision to leak into the current one, resulting in erratic and difficult-to-debug AI behavior.

- **Cross-NPC Sharing:** Never share a single EvaluationContext instance across multiple NPCs that are being evaluated concurrently. Each NPC's evaluation is a distinct operation and requires its own isolated context.

- **Long-Term Storage:** This object represents a transient configuration. Do not serialize it or store it as part of an NPC's persistent state. The parameters it holds are for an immediate, point-in-time decision.

## Data Pipeline
The EvaluationContext does not process data itself; rather, it provides the parameters that guide the data processing pipeline of the AI Evaluator.

> Flow:
> AI Controller -> **Creates & Configures EvaluationContext** -> Passes to Evaluator -> Evaluator uses context to filter and score potential Decisions -> A final Decision is selected and executed.

