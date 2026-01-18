---
description: Architectural reference for MatchResult
---

# MatchResult

**Package:** com.hypixel.hytale.server.core.command.system
**Type:** Transient

## Definition
```java
// Signature
public class MatchResult implements Comparable<MatchResult> {
```

## Architecture & Concepts
The MatchResult class is an immutable value object that quantifies the quality of a string comparison within the server's command processing system. It is not merely a simple score; it is a multi-dimensional metric used to deterministically select the best possible command or argument match from a set of candidates based on user input.

Its primary role is to serve as the core comparison mechanism for command dispatching and tab-completion suggestions. By encapsulating several aspects of a match—such as which part of a command was matched (name, alias), its depth in the command tree, and the raw string similarity—it provides a sophisticated and hierarchical sorting capability. This prevents ambiguity and ensures that the most relevant command is always executed.

The class implements the Comparable interface, allowing collections of MatchResult objects to be sorted efficiently to find the single best outcome. The static constants, NONE and EXACT, act as sentinel values to represent the worst and best possible outcomes, simplifying comparison logic.

## Lifecycle & Ownership
- **Creation:** Instances are created ephemerally by the command parsing system. The static factory method, MatchResult.of, is the intended entry point, which calculates the Levenshtein distance between the input and a candidate string.
- **Scope:** Extremely short-lived. A MatchResult object typically exists only for the duration of a single command resolution operation. It is created, compared against other MatchResult instances, and then becomes eligible for garbage collection.
- **Destruction:** Managed entirely by the Java garbage collector. As a simple value object with no external resources, it requires no explicit cleanup.

## Internal State & Concurrency
- **State:** Immutable. All internal fields (term, depth, type, match) are final and are set exclusively at construction time. The object's state cannot be modified after creation.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. Instances can be safely shared, passed, and compared across multiple threads without any need for external synchronization or locks. This is critical for a server environment where command processing may occur on multiple worker threads.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| of(termDepth, depth, type, text, search) | MatchResult | O(N*M) | **Primary Factory.** Creates a new MatchResult. Complexity is dominated by the internal Levenshtein distance calculation. |
| min(other) | MatchResult | O(1) | Compares this instance to another and returns the "better" match based on the internal hierarchical comparison logic. |
| compareTo(o) | int | O(1) | Implements the Comparable interface. Provides a strict ordering based on term, type, depth, and finally match score. |

## Integration Patterns

### Standard Usage
MatchResult is used by the command system to evaluate and rank potential command candidates against user input. The system generates a result for each candidate and uses the comparison logic to find the best one.

```java
// A command resolver evaluating two potential matches for user input "pl"
MatchResult playerMatch = MatchResult.of(0, 0, MatchResult.NAME, "player", "pl");
MatchResult pluginMatch = MatchResult.of(0, 0, MatchResult.NAME, "plugin", "pl");

// The resolver finds the best match using the built-in comparison
MatchResult bestMatch = playerMatch.min(pluginMatch);

// The command corresponding to bestMatch is then selected for execution.
```

### Anti-Patterns (Do NOT do this)
- **Misinterpreting the Score:** Do not assume the integer returned by getMatch (the Levenshtein distance) is the sole indicator of quality. The comparison logic is hierarchical; term, type, and depth are evaluated first. A less-similar string match can be chosen over a more-similar one if it has a better type or depth.
- **Ignoring Sentinel Values:** Do not write custom logic to check for "no match". Always compare against the static MatchResult.NONE instance for a definitive result.

## Data Pipeline
The MatchResult class is a critical transformation step in the command processing data pipeline. It converts raw string inputs into a quantifiable and comparable metric that enables deterministic command resolution.

> Flow:
> Raw User Input String -> Command System -> **MatchResult.of()** for each candidate -> Collection of MatchResult objects -> Sorting via **compareTo()** / **min()** -> The single best Command object

