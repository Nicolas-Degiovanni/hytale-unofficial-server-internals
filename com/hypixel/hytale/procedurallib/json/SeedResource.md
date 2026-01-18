---
description: Architectural reference for SeedResource
---

# SeedResource

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Mixin Interface

## Definition
```java
// Signature
public interface SeedResource {
```

## Architecture & Concepts
The SeedResource interface is not a standalone component but rather a foundational contract or **mixin** for any class participating in the procedural generation pipeline. Its primary architectural role is to provide a standardized, decentralized mechanism for accessing shared resources and reporting diagnostic information.

By implementing this interface, a class gains two core capabilities:
1.  **Shared Buffer Access:** It provides direct, convenient access to shared, and likely thread-local, `ResultBuffer` instances. This pattern decouples generation logic from the complexities of buffer management, preventing the need to pass buffer references through deep and complex call stacks.
2.  **Seed Reporting:** It establishes a common, overridable framework for logging the seed values used during generation. This is critical for debugging and reproducing specific procedural generation outcomes.

Classes that perform discrete generation steps, such as nodes within a procedural graph, are the primary implementors of this interface. It effectively acts as a "resource handle" embedded within the logic that needs it.

### Lifecycle & Ownership
As an interface, SeedResource has no lifecycle of its own. Its lifecycle is entirely dependent on the lifecycle of the object that implements it.

-   **Creation:** An object implementing SeedResource is created by its owner, typically a procedural graph executor or a factory responsible for instantiating generation nodes. The interface itself is never instantiated.
-   **Scope:** The scope is identical to that of the implementing object. For a transient generation task, the object may be short-lived. For a persistent world generator component, it may exist for the entire server session.
-   **Destruction:** The interface requires no explicit cleanup. It is garbage collected along with its implementing object.

## Internal State & Concurrency
-   **State:** The SeedResource interface is **stateless**. It holds no fields and its default methods do not modify any internal state. It serves as a gateway to external, potentially stateful resources like the static fields in the ResultBuffer class.

-   **Thread Safety:** The interface is inherently **thread-safe**. However, the resources it provides access to may have their own concurrency constraints. The `ResultBuffer` instances are expected to be thread-local to support parallelized world generation. The default implementation of `writeSeedReport` uses `System.out.println`, which is a synchronized, thread-safe operation.

    **WARNING:** While `System.out.println` is thread-safe, its use in a heavily multi-threaded generator can lead to interleaved and unreadable log output. Production-ready implementors **must** override `writeSeedReport` to delegate to a structured, thread-safe logging framework.

## API Surface
The public contract consists entirely of default methods, designed to be used directly or overridden for specialized behavior.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| localBounds2d() | ResultBuffer.Bounds2d | O(1) | Provides access to a shared 2D bounds buffer. |
| localBuffer2d() | ResultBuffer.ResultBuffer2d | O(1) | Provides access to a shared 2D result buffer. |
| localBuffer3d() | ResultBuffer.ResultBuffer3d | O(1) | Provides access to a shared 3D result buffer. |
| shouldReportSeeds() | boolean | O(1) | Determines if seed reporting is enabled. Returns false by default. |
| reportSeeds(int, String, String, String) | void | O(1) | Formats and reports a seed value if `shouldReportSeeds` is true. |
| writeSeedReport(String) | void | O(1) | The final output sink for a seed report. Defaults to System.out. |

## Integration Patterns

### Standard Usage
A class implementing a specific procedural generation algorithm will implement this interface. It may override `shouldReportSeeds` to enable debugging for that specific step.

```java
// A node in a procedural graph implements the interface
public class BiomePlacementNode implements SeedResource {

    @Override
    public boolean shouldReportSeeds() {
        // Enable reporting for this specific node for debugging
        return true;
    }

    @Override
    public void writeSeedReport(String seedReport) {
        // Route to a proper logger instead of System.out
        WorldGenLogger.debug(seedReport);
    }



    public void execute(GenerationContext context) {
        // 1. Get a shared buffer without needing it passed in
        ResultBuffer.ResultBuffer2d buffer = this.localBuffer2d();

        // 2. Perform generation work...
        int finalSeed = context.getSeed() ^ 0xABCD;

        // 3. Report the seed value used in this step
        this.reportSeeds(finalSeed, "base_seed", "biome_placement", null);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Ignoring Overrides:** Relying on the default `writeSeedReport` in a production environment will pollute standard output and offers no structured logging capabilities. This method is intended to be overridden.
-   **Long-Term Storage of Buffers:** Do not store the `ResultBuffer` instances returned by the `localBuffer` methods in long-lived fields. These buffers are designed for transient, per-operation use and their state is not guaranteed between invocations.
-   **Misunderstanding the Contract:** This interface is for accessing shared resources. Do not add state or complex logic to your implementation of its methods. The implementation should be a simple delegation to a logging system or a configuration check.

## Data Pipeline
SeedResource does not process data itself; it is a utility that provides resources *to* a data pipeline. Its role is to inject shared buffers and provide a side-channel for diagnostic output.

**Buffer Provisioning Flow:**
> Generation Executor -> Invokes `execute()` on an implementing class -> `localBuffer2d()` is called -> **SeedResource** returns shared buffer -> Implementing class writes procedural data into the buffer.

**Seed Reporting Flow:**
> Implementing class calculates a seed -> `reportSeeds()` is called -> **SeedResource** checks `shouldReportSeeds()` -> **SeedResource** formats the report string -> `writeSeedReport()` is called -> Log Sink (e.g., Logger, Console).

