# fast-surface-nets

A fast, chunk-friendly implementation of Naive Surface Nets on regular grids.

Surface Nets is an algorithm for extracting an isosurface mesh from a [signed distance
field]((https://en.wikipedia.org/wiki/Signed_distance_function)) sampled on a regular grid. It is nearly the same as Dual
Contouring, but instead of using hermite (derivative) data to estimate surface points, Surface Nets will do a simpler form
of interpolation (average) between points where the isosurface crosses voxel cube edges.

Benchmarks show that [`surface_nets`] generates about 20 million triangles per second on a single core of a 2.5 GHz Intel
Core i7. This implementation achieves high performance by using small lookup tables and SIMD acceleration provided by `glam`
when doing 3D floating point vector math.

High-quality surface normals are estimated by:

1. calculating SDF derivatives using central differencing
2. using bilinear interpolation of SDF derivatives along voxel cube edges

When working with sparse data sets, [`surface_nets`] can generate meshes for array chunks that fit together seamlessly. This
works because faces are not generated on the positive boundaries of a chunk. One must only apply a translation of the mesh
into proper world coordinates for given chunk.

## Example Code

```rust
use fast_surface_nets::ndshape::{ConstShape, ConstShape3u32};
use fast_surface_nets::{surface_nets, SurfaceNetsBuffer};

pub type ChunkShape = ConstShape3u32<18, 18, 18>;

// This SDF should have a single octant of a sphere for an isosurface.
let mut sdf = [1.0; ChunkShape::SIZE as usize];
for i in 0u32..(ChunkShape::SIZE) {
    let [x, y, z] = ChunkShape::delinearize(i);
    sdf[i as usize] = (x * x + y * y + z * z) as f32 - 15.0;
}

let mut buffer = SurfaceNetsBuffer::default();
surface_nets(&sdf, &ChunkShape {}, [0; 3], [17; 3], &mut buffer);

assert_eq!(buffer.indices.len() / 3, 48);
```