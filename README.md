![Raygon Logo][logo] Raygon
===========================

Raygon is a WIP high performance CPU path tracer written in the Rust programming language. It will feature state of the art light transport integrators including path tracing, bidirectional path tracing and VCM. Perhaps even more in the future.

[Join our Discord Server](https://discord.gg/Y54gQxH) and consider [Supporting Raygon on Patreon](https://www.patreon.com/raygon)

### Example

Here is the latest rendered image created simply for the purposes of testing. Its quality is not indicative of the final product, but it shows how far along the project is.
![Demo][latest_demo]

And an Ambient Occlusion demo used for the [Information](#Information) section
![AO Demo][ao_demo]

## Current Features

### Performance

Much effort has been put into creating the fastest code possible. Raygon uses a custom hand-written linear algebra library based on `packed_simd`, making use of explicit SIMD wherever possible. Final binaries will be precompiled for specific architectures to make full use of SSE and AVX instruction sets on modern hardware.

Currently, Raygon just uses a regular LBVH generated with a simple SAH heuristic, which achieves performance up to 18 Million rays per second on 29 threads with my AMD Threadripper 1950X, or around 620,000 rays/second per thread. While less than what a GPU can achieve for brute-force path tracing, Raygon will support complex light transports that can't be run on a GPU, making this performance advantageous over competing renderers.

### Materials

Raygon features a lightweight virtual machine for arbitrary material shader evaluation, allowing for complex procedural materials easily on-par with Blender Cycles, and vastly outclassing other industry renderers.

### Film

Raygon renders to a densely packed multi-channel film, with support for over 20 render channels such as Emission, World Normal, Direct and Indirection light, Roughness, Albedo, various object and material IDs, ThreadId, and so forth.

It also supports using unique direct/indirect light channels for different light groups, allowing the separation of different light sources and the combination of such in post-production. This can be used to tweak light emission intensity and color after the render has completed. Don't like a light? Turn it off in post.

### Camera

Currently, Raygon supports only a thin-lens perspective and orthographic camera capable of Depth of Field and time integration/animated transforms. It also supports a bladed aperture of my own design, rather than a perfectly circular aperture. Demo images will be available soon.

Equirectangular, Stereoscopic Equirectangular (VR) and complex realistic cameras using real-world lens descriptions are also planned.

In the future, it may be possible to write custom camera scripts for more complex use cases such as baking renders.

### Primitives

Raygon supports triangles/triangle meshes, cubic Bézier splines, and perfect spheres (for lights and particles).

Additionally, Raygon will support signed distance field shapes, allowing for infinitely detailed procedural shapes not limited to complex mathematical shapes and fractals.

### Animation

Raygon already partially supports integrating through time in the form of animated transformations, allowing for accurate motion blur of shapes, lights and cameras. Anything and everything can have motion.

### Information

Raygon has been designed to have anything and everything report its memory usage. Almost every single byte is accounted for and can be printed in the debug modes.

Here is the debug output from rendering the Ambient Occlusion example, rendered with a debug build.
![Debug Log][debug_log]

This particular run did not have the profiler enabled, so times show 0ns

You can even see a few debug logs from the material virtual machine optimizer.

## Planned Features (not extensive)

* Bidirectional Path Tracing and VCM (Vertex Connection and Merging)
* Path Space Regularization
* Plugins
    * Blender, Houdini, Amethyst, etc.
    * Provide a public C API for anyone
    * Provide a agnostic network API for anyone
* LMBVH4 for even faster ray tracing with SIMD
* Many modern materials such as multi-scatter GGX
* Efficient layered materials
* Material shader JIT compilation
    * Would possibly allow for Turing-complete shaders with complex control flow
* Much more.

[logo]: ./assets/logo48.png "Raygon Logo"
[latest_demo]: ./assets/test34.png "Latest test render"
[ao_demo]: ./assets/test35.png "AO Demo"
[debug_log]: ./assets/debug_log.png "Debug Log"