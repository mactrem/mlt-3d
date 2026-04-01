# MapLibre Tile (MLT) 3D Extension

This design document builds on the [initial proposal](https://github.com/alasram/mlt_3d_tile_schema) for MLT 3D
by [alasram](https://github.com/alasram).


## Context and scope

The MLT 3D extension aims to provide a simplified 3D tile format optimized for the cartographic use case—enabling
low-latency rendering that preserves the familiar "slippy map" experience, even on mobile devices and
bandwidth-constrained networks. The target is a simplified 3D representation of the real-world environment,
specifically navigation views in the style of [Google Maps Immersive Navigation](https://blog.google/products-and-platforms/products/maps/ask-maps-immersive-navigation/), rather than LiDAR point clouds or photorealistic 3D with high-resolution textures such as Google's [Photorealistic 3D Tiles](https://developers.google.com/maps/documentation/tile/3d-tiles) (where Gaussian splatting is a promising direction). That photorealistic segment is already well served by 3D Tiles—a powerful format and the right choice massive 3D geodata content such as photogrammetry, BIM/CAD and point clouds. However, for navigation-oriented 3D mapping, 3D Tiles introduces unnecessary overhead through concepts designed for the general case: the genericity of glTF, hierarchical levels of detail with geometric error, different refinement strategies, adaptive spatial data structures. The goal of MLT 3D is therefore to provide a streamlined alternative that targets the cartographic use case directly, building on MLT's core concepts (columnar layout, encodings, ...).


## Goals and non-goals

### Goals

#### Functional requirements 
The format should support the encoding of 2.5 and 3D objects such as
- 3D lanes including lane changes and pedestrian crossings
- 3D objects such buildings (LOD1 and LOD2), bridges
- 3D instance objects such as traffic lights, stop signs, trees
- 2.5D terrain

High resolution 3D buildings (LOD3+) are only used occasionally for known and dominant buildings as orientation
in the navigation workflow but not widely used in the scene.

#### Non-functional requirements 
- Optimized for minimal tile size to perform well in bandwidth-constrained and resource-limited environments
- Preserve the familiar "slippy map" user experience by enabling low-latency rendering


### Non Goals

Photorealistic simulation of the environment, including concepts for efficiently rendering complex, highly detailed 3D models with high-resolution textures.


## Design

### Ideas

- parametric first instead of meshes where possible
- use compute/mesh shader for extrusion where possible and avoid storing (larger) pre-computed information  
  (Space–time tradeoff)

### Encoding

#### FeatureTable (ExtrusionTable)

Stores parametric/procedural 2.5D/3D objects as compact attribute descriptions rather than pre-tessellated meshes. This is the preferred strategy over GPUTables wherever applicable, as it minimizes tile size—and therefore latency—by deferring mesh generation to the client via compute or mesh shaders.

**Core paradigm:** compact representation → GPU expansion → render.

- **Primary use case:** 2.5D objects such as LOD1 extruded buildings and 3D roads, represented as parametric data.
- **Modeling approach:** procedural/parametric geometry where the mesh is reconstructed on the GPU from compact
  attribute descriptions, shifting away from the glTF model of storing fully pre-tessellated geometry.
- **Memory layout:** struct packing for vertex buffers (SoA)—compress each column individually,
  then interleave into a single stream (similar to the approach described in the Lance paper).

Example 3D Object Types:
  - Buildings (LOD1)
  - 3D Roads
    - similar idea then [OpenDRIVE](https://www.asam.net/standards/detail/opendrive/) where a road network is modelled along a reference line
    - Linear Referencing to combine vertices with attributes to specify lane width
    - No styling information included in the FeatureTable only via style document based on data driven styling
    
Open Questions
    - How to encode linear referencing? -> int vs double for ranges
    - Only one component of uv for FeatureTable and also data type u32 to be compatible with WebGPU (u is distance along line)?
    - How to map textures to lanes? -> are side stripes and sidewalks only textures (SDFs) or vector data for sharpness
    - No support for colors? -> only data-driven styling via style document?
    - Only encode tangent?
    - Only basically support LOD1 buildings and roads as 3D objects in a FeatureTable?
    - How to reference textures?
    - Use AoS vs SoA approach for VertexBuffer? -> Nvidia recommends AoS
    - Also describe LOD2 buildings (extrusion and roof types) and many bridges parametric? 
    - Allow to batch extruded cubes to be merged into a single feature?
  

![Feature Table Layout ](diagrams/feature_table.png)

#### GPUTables 
Stores explicit 3D mesh geometry ready for direct GPU consumption, used when parametric representation
via FeatureTable is not feasible.

- PrimitiveTable
  - Contains the explicit meshs geometries in a GPU-ready layout
  - Only support mesh topologies? 
  - AoS vs SoA for the VertexBuffer? -> Use interleaved layout for VertexBuffer because delta encoding can be applied independently on each component
- ObjectTable
  - include a human-readable name per object?
- SceneTable/MeshTable
  - Acts as the 3D counterpart to a FeatureTable—same columnar structure, but instead of inline geometry
columns it references object instances via ID and attaches a per-instance transform


![GPUTables](diagrams/gpu_tables.png)

- TerrainTable
- MaterialTable

#### Assets

- Texture Atlas (Texture Container File)
- Instance Container File


## Data
- Overture Maps -> https://docs.overturemaps.org/guides/transportation/segments-and-connectors/
  - buildings
  - transportation
    - Segment Z-order -> https://docs.overturemaps.org/guides/transportation/segments-and-connectors/#level-z-order
    - Linear referencing -> https://docs.overturemaps.org/guides/transportation/linear-referencing/
  

## Open Questions

- Which refinement strategy should be used for 3D objects such as buildings (replacement only, or also additive)?
- Which data types should be used, and which graphics API constraints apply? -> For example, `u16` is not supported in WebGPU
- Is a data structure needed at higher zoom levels (e.g., z15+) to signal which tiles contain only 3D model data in dense areas and no vector data similar to the availability structure in 3D Tiles?
- How should textures for linestring geometries such as roads be handled efficiently?
- Can point-based positioning for terrain-clamped objects (no scaling, no rotation) be combined with modle transform-matrix-based positioning for non-clamped objects to reduce storage size?
- Are normal maps actually needed in the first iteration?
- MLT currently supports only linear interpolation between vertices, not splines or Bézier curves, which are commonly used in road modeling. Should these geometry types be added to MLT?
- How should an instanced model in the SceneTable reference its corresponding entry in the instance container file?
- Where can test data for 3D roads and LOD2+ buildings be sourced?









