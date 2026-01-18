---
description: Architectural reference for StringCompareUtil
---

# StringCompareUtil

**Package:** com.hypixel.hytale.common.util
**Type:** Utility

## Definition
```java
// Signature
public class StringCompareUtil {
```

## Architecture & Concepts
StringCompareUtil is a foundational, stateless utility class that provides a centralized collection of advanced string comparison and distance algorithms. Residing in the `common.util` package, its functions are universally accessible to both client and server-side logic, ensuring consistent behavior for features that rely on textual analysis.

This class embodies the principle of a pure functional component. Each method operates exclusively on its input arguments to produce a result, with no side effects or reliance on external state. Its primary role is to abstract complex and computationally intensive algorithms, such as Levenshtein distance and a proprietary fuzzy matching score, away from higher-level business logic. It is a critical dependency for systems like in-game search, command suggestion engines, and content filtering services.

## Lifecycle & Ownership
As a static utility class, StringCompareUtil does not have a traditional object lifecycle.

- **Creation:** The class is loaded into the JVM by the system ClassLoader upon its first use. It is never instantiated; direct construction is not possible as it lacks a public constructor and all methods are static.
- **Scope:** Application-level. The class and its methods are available for the entire duration of the application's runtime.
- **Destruction:** The class is unloaded from the JVM when the application terminates. There are no instances to garbage collect or resources to release.

## Internal State & Concurrency
- **State:** This class is completely **stateless** and **immutable**. It contains no member fields and does not cache any data between method invocations. All computations are performed within the local scope of each static method call.
- **Thread Safety:** StringCompareUtil is inherently **thread-safe**. Due to its stateless nature, its methods can be invoked concurrently from any number of threads without risk of data corruption or race conditions. No external synchronization is required by the caller.

## API Surface
The public contract consists of pure functions for string analysis.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| indexOfDifference(cs1, cs2) | int | O(min(N, M)) | Returns the first index at which two CharSequences differ. Returns -1 if they are identical. |
| getFuzzyDistance(term, query, locale) | int | O(N * M) | Calculates a proprietary fuzzy match score. Higher scores indicate a better match. Rewards consecutive character matches heavily. |
| getLevenshteinDistance(s, t) | int | O(N * M) | Computes the classic Levenshtein distance between two strings, representing the minimum number of single-character edits required to change one string into the other. |

**WARNING:** The fuzzy and Levenshtein distance methods have a time complexity of O(N * M), where N and M are the lengths of the input strings. Avoid using them with very large strings in performance-critical loops.

## Integration Patterns

### Standard Usage
This class should be used via static method calls for any logic requiring non-trivial string comparison, such as implementing a search or autocomplete feature.

```java
// Example: Finding the best match for a search query from a list of items.
String query = "hytl";
List<String> items = List.of("Hytale", "Hytale Blog", "Update");
String bestMatch = null;
int lowestDistance = Integer.MAX_VALUE;

for (String item : items) {
    int distance = StringCompareUtil.getLevenshteinDistance(query, item.toLowerCase());
    if (distance < lowestDistance) {
        lowestDistance = distance;
        bestMatch = item;
    }
}
// bestMatch is now "Hytale"
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never attempt to create an instance of this class (e.g., `new StringCompareUtil()`). It provides no value and is not the intended use pattern.
- **Null Argument Violations:** Do not pass null values to methods annotated with Nonnull. This will result in an immediate `IllegalArgumentException` and crash the calling thread. Always perform null checks before invoking these methods.
- **Misinterpreting Fuzzy Score:** The `getFuzzyDistance` method does not return a standardized metric. Its scoring is specific to Hytale's needs. Do not assume it behaves like Levenshtein or other standard algorithms; its results should be calibrated for the specific feature it is used in.

## Data Pipeline
StringCompareUtil is not a pipeline itself but a critical processing stage within a larger data flow. It typically acts as a "scorer" or "filter" in systems that handle user-provided text.

> **Example Flow: In-Game Search**
>
> User Input String -> Search Service -> **StringCompareUtil.getFuzzyDistance** -> Scored & Sorted Result Set -> UI Rendering System

