---
description: Architectural reference for MeshVoxelizer
---

# MeshVoxelizer

**Package:** com.hypixel.hytale.builtin.buildertools.objimport
**Type:** Utility

## Definition
```java
// Signature
public final class MeshVoxelizer {
    // Note: The constructor is private. This class cannot be instantiated.

    public record VoxelResult(boolean[][][] voxels, @Nullable int[][][] blockIds, int sizeX, int sizeY, int sizeZ) {
        // ... methods for querying the result
    }
}
```

## Architecture & Concepts
The **MeshVoxelizer** is a stateless computational geometry utility that serves as a critical bridge between continuous polygonal geometry and the discrete voxel world of Hytale. Its primary function is to convert a 3D model, represented as an **ObjParser.ObjMesh**, into a three-dimensional grid of blocks. This process, known as voxelization, is fundamental to the in-game builder tools that support importing external 3D assets.

The voxelization pipeline is a multi-stage process executed within a single static method call:

1.  **Scaling & Normalization:** The input mesh is uniformly scaled so that its height matches the specified *targetHeight*. The system can either preserve the model's original coordinate origin relative to the scaled dimensions or normalize its position so that its bounding box starts at (1,1,1) in the temporary grid. This ensures predictable dimensions and placement.

2.  **Surface Rasterization:** The core of the algorithm iterates through each triangular face of the scaled mesh. It rasterizes the triangle's edges and interior into a temporary 3D boolean grid, forming a hollow, one-voxel-thick "shell" of the model.

3.  **Material & Texture Evaluation:** During rasterization, if material and texture data are provided, the system determines the appropriate block ID for each surface voxel. This can be a direct mapping from a material name to a block ID or a more complex process involving sampling the material's texture at the corresponding UV coordinate, converting the sampled color to a Hytale block via the **BlockColorIndex**, and assigning that block's ID.

4.  **Solid Filling (Optional):** If the *fillSolid* flag is enabled, the system performs a 3D flood-fill algorithm. It starts from the outside of the temporary grid and marks all reachable empty voxels as "exterior". Any unmarked voxels inside the rasterized shell are considered "interior" and are filled in.
    *   **Warning:** This process assumes the input mesh is watertight. Gaps or holes in the mesh will cause the flood-fill to "leak" into the interior, resulting in a hollow or incorrectly filled model.

5.  **Result Cropping:** Finally, the large temporary grid is analyzed to find the minimal bounding box that contains all solid voxels. The final result is cropped to these dimensions to produce a compact and efficient data structure, returned as a **VoxelResult**.

## Lifecycle & Ownership
-   **Creation:** The **MeshVoxelizer** is a pure utility class with a private constructor. It is never instantiated.
-   **Scope:** The class has no state and therefore no lifecycle. Its functionality is exposed through static methods, and its operational scope is confined entirely to the duration of a single `voxelize` method call. All intermediate data structures, such as voxel grids, are allocated on the heap and become eligible for garbage collection as soon as the method returns the final **VoxelResult**.
-   **Destruction:** Not applicable.

## Internal State & Concurrency
-   **State:** The **MeshVoxelizer** class is completely stateless. It holds no member variables and all computations are performed on data passed in as method arguments. All state required for the operation (e.g., the `shell` and `blockIds` arrays) is created, mutated, and discarded within the scope of a single method call.

-   **Thread Safety:** This class is unconditionally thread-safe. Its stateless, functional design ensures that concurrent calls to the `voxelize` method with different inputs will not interfere with each other. It is safe to use in multi-threaded environments, for example, to process multiple model imports in parallel on a background thread pool.

## API Surface
The public API consists of a series of overloaded static `voxelize` methods. The most comprehensive signature is detailed below.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| voxelize(mesh, targetHeight, fillSolid, materialTextures, materialToBlockId, colorIndex, defaultBlockId, preserveOrigin) | VoxelResult | High - O(W\*H\*D) | The primary entry point. Converts a mesh to a voxel grid. Complexity is dominated by the volume of the intermediate grid (Width \* Height \* Depth). Throws NullPointerException if mesh is null. |

## Integration Patterns

### Standard Usage
The typical use case involves parsing an OBJ file, preparing material and texture maps, and then invoking the voxelizer to get a result that can be used to place blocks in the world.

```java
// Assume 'mesh' is a loaded ObjParser.ObjMesh
// Assume 'materialTextures' is a Map<String, BufferedImage>
// Assume 'colorIndex' is an initialized BlockColorIndex

int desiredHeight = 64;
boolean makeSolid = true;

// Voxelize the mesh into a grid of blocks
MeshVoxelizer.VoxelResult result = MeshVoxelizer.voxelize(
    mesh,
    desiredHeight,
    makeSolid,
    materialTextures,
    null, // No direct material-to-ID map
    colorIndex,
    1, // Default block ID (e.g., stone)
    false // Normalize origin
);

// The 'result' can now be used to place blocks in the world
// for (int x = 0; x < result.sizeX(); x++) { ... }
```

### Anti-Patterns (Do NOT do this)
-   **Processing Massive Inputs on Main Thread:** The voxelization algorithm is computationally expensive and memory-intensive. Passing a high-polygon mesh or a very large `targetHeight` can result in the allocation of enormous temporary arrays, potentially causing an OutOfMemoryError and blocking the calling thread for a significant amount of time. **Always** run this process on a background worker thread.
-   **Assuming Watertight Meshes:** Enabling `fillSolid` on a mesh with holes will produce incorrect results, often appearing hollow or partially filled. The input mesh must be a closed, manifold surface for the flood-fill algorithm to work correctly.
-   **Ignoring Memory Implications:** A `targetHeight` of 256 for a roughly cubic model will create temporary grids exceeding 258x258x258 voxels. Be mindful of the memory footprint, which grows cubically with the target dimension.

## Data Pipeline
The **MeshVoxelizer** acts as a pure transformation function within a larger data processing pipeline for model importing.

> Flow:
> ObjParser.ObjMesh -> **MeshVoxelizer**.voxelize() -> **VoxelResult** (containing voxel and block ID grids) -> World Building System (places blocks)

