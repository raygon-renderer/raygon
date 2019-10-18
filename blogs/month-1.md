This Month in Raygon 1
========================

It's been about 51 days since I first announced Raygon, a WIP high-performance CPU path tracer written in the Rust programming language, which will feature state of the art light transport integrators, including path tracing, bidirectional path tracing and VCM.

In this post I'll go over some of the features implemented or improved, and show some recent renders, then talk about the future of Raygon and how you can help.

## Look Back

Before we get into the updates, it's important to remember where the project was at on August 24th:

<details>
<summary>
Click to expand the old demo image
</summary>

![Old Demo][old_demo]
</details>

This image was rendered only a couple weeks after getting triangles to the screen at all. Much has changed since then.

# Current Status

In the past couple months I've implemented almost the entire path tracer. Physically based textures/procedural materials, light sampling, environment sampling, and more:

* Full path tracing
* Pixel [Filter Importance Sampling](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.183.3579&rep=rep1&type=pdf) (FIS)
    * All renders shown below use a Blackman-Harris filter
    * FIS is generally equal or better quality than traditional splatting, while being incredibly easy to implement and integrate into highly concurrent rendering systems like Raygon.
* Physically based materials
    * GGX Microfacet distribution for specular and Oren-Nayar for diffuse
        * About halfway through implementing multiscatter GGX for even more accurate materials
    * Uses the relatively recent [GGX Visible-Normal Distribution Function](http://jcgt.org/published/0007/04/01/paper.pdf)
        * i.e. it doesn't sample directions that are impossible and focuses on important areas
* Basic Texture sampling
    * Textures are lazily loaded as materials are encountered
    * LDR and HDR images are supported, in almost any format the `image` crate can read.
        * sRGB->Linear is taken care of by the shader system, decoding instructions are inserted if necessary/desired
    * Textures are deduplicated and shared among materials
* Auto-generated materials that include:
    * Albedo (with optional alpha-masking)
    * Alpha masks
        * Used in some of the leaves materials
    * Emission
        * Emissive surfaces are automatically considered light sources
        * Raygon doesn't use analytic lights, only emissive primitives
    * Roughness
    * Metalness
    * Baked ambient-occlusion
    * Tangent-space normal mapping
* Direct Light Sampling/Next Event Estimation with multiple importance sampling
    * BxDF MIS is integrated into bounce logic, so no rays are wasted (unlike PBRTv3)
    * Surface-area based importance sampling of many area lights
        * Will replace later with a full light-tree and BxDF-based sampling
    * Spherical sampling of individual triangle lights
        * Technically a form of importance sampling
* Some basic tonemapping, although eventually I want to implement full ACES
* Environment map importance sampling for HDRi lighting
* Shadow terminator fix for low-poly meshes
    * Needs to opt into as it only really works well for closed meshes
    * Works well with instancing using a robust object "path" solution to detect self-shadowing
* Spectra-agnostic render pipeline
    * Even though Raygon doesn't have spectral rendering itself implemented yet, all BxDFs and integrators are spectra-agnostic using traits, meaning it should be easy to drop in.
* Highly optimized SIMD and Scalar math routines
    * Uses new `simd` intrinsics where possible, and robust approximations otherwise.
* Efficient low-overhead profiling tools

# Renders

Though, the best way to show where the project is at today is visually:

The same living room now:
![Test 198][test218]

Here is an extreme stress-test for object instancing with over **65 Billion** effective triangles, and also using an HDRi with environment importance sampling. Some of the tree leaves also used alpha-masked materials for greater detail.
![Test 216][test216]

Cornell-like Box with the classic Dragon and Buddha
![Test 173][test173]

And here, just a few hours before writing this, I added initial support for physically based metals using conductor Fresnel effects for their color. This one also uses an importance-sampled HDRi for lighting.
![Test 213][test213]

# The Near Future

Support Raygon on [Patreon](https://www.patreon.com/raygon) or [Ko-Fi](https://www.ko-fi.com/raygon). All donations count torwards licenses when Raygon eventually fully releases. Furthermore, donating just $30 will give you a lifetime personal license, and $50 will get your name featured in the executable itself.

Feel free to [Join our Discord Server](https://discord.gg/Y54gQxH) as well.

Now then, with that out of the way, onto the real meat and potatoes.

## Planned Features

These aren't in any particular order.

### Spectral Rendering

One of the most important features in the works right now is spectral rendering, which will allow for reflective/refractive dispersion, thin-film interference, perfect blackbody radiation,  fluorescence, and polarization. My implementation will be based on the Hero-wavelength method, with some new advancements in color upsampling.

### Bidirectional Path Tracing and SPPM/VCM

With the unidirectional path tracer mostly complete at this point aside from volumes, in the next month or so work will begin on the bidirectional path tracer, and shortly after that Stochastic Progressive Photon Mapping. With those two implemented, VCM (Vertex Connection and Merging) can be built off of that, which combines the best of both in a relatively simple integrator that is remarkably effective.

### Scene Description Format(s)

Currently, scenes are either procedurally generated (such as the grass and trees render) or loaded from `.obj`/`.mtl` files, with materials being generated from the `mtl`.

The next step is to create a formal scene description format for anything and everything Raygon can render, in a way that is both human and machine readable/writable. Nested descriptions across multiple files will be supported to build up complex scenes of manual and procedural scenes.

### Websocket API

Of course, a file format alone isn't sufficient for interactive rendering from a plugin such as in Blender. To that end, the next step is to create a websocket-based API for communicating assets and rendered images (final and in-progress pixels/tiles) between Raygon and plugins. Perhaps even an extra small web interface for monitoring render progress in realtime.

## Refinement of Existing Features

There is always new research being published, and I want to incorperate the latest graphics science into Raygon.

### Transmission, Transparency and Translucency

Transmission is planned immediately after this update blog, and is already in progress. Special support for "thin" objects is also being worked on, allowing for something like thin glass windows to be simulated using only a single plane or one-sided mesh, rather than multiple layers of geometry.

### [Multiscatter GGX](https://eheitzresearch.wordpress.com/240-2/)

This is a fairly high priority feature that needs to be implemented as soon as possible. It gives much greater realisism to rough surfaces and especially rough glass, as it does not lose energy from ignored scattering events in the material surface.

### Better Samplers and Samples

Currently, Raygon is just using a plain old dumb random sampler for everything. Random samples are about as bad as it gets.

This was chosen originally because for one it's incredibly easy to implement and use, and second because traditional stratified samplers were not initially possible given some of the other features I've implemented (i.e. material shaders can sample random numbers, adaptive number of samplers per pixel, etc.).

However, very recently a [new algorithm](https://graphics.pixar.com/library/ProgressiveMultiJitteredSampling/) has come to my attention for *progressively* generating stratified samples, which is ideal for Raygon. I plan to implement this very soon, and doing so should reduce noise in renders drastically.

### Textures

Basic textures are implemented, and the next step is to implement MIP mapping and anisotropic trilinear filtering. Additionally, normal maps will ideally be compressed to use only two texture channels (as they are unit vectors, it's very easy to recalculate a missing component) to save space.

Long-term goals with textures are to implement a megatexture-like streaming system for transparently loading and unloading parts of textures as they are used, which can drastically reduce core memory usage.

### Deformation with Motion Blur

Procedural vertex shaders with displacement and mesh deformation are planned, with support for stochastic ray-traced motion blur. Animated fluids, morphs and skeletal meshes will be made possible by this.

## Long-Term Plans

### WebAssembly

With Rust's WASM support coming along nicely, it may eventually be possible to compile the entirety of Raygon for use in a web browser as an easy to use networked render node with near-native performance.

Although, as it won't be perfectly native performance, especially if WASM SIMD support isn't added by then, a couple of such render nodes will be provided for free for all license tiers, and probably only available on a local network given the bandwidth requirements.

### Heterogeneous Volumes

Heterogeneous Volumes are a difficult problem in Rust, though not for any limitation in Rust itself. OpenVDB is the industry standard method for representing such volumes, but sadly OpenVDB is written in C++. It will take time to either bridge that gap efficiently or reimplement the necessary parts for rendering.

### Denoiser Integration

[Bayesian Collaborative Denoising](https://github.com/superboubek/bcd) will be implemented and used as the default denoiser for final beauty images. However, if you would prefer something like Intel's OIDN or Nvidia OptiX, Raygon will easily be able to export Deep Image render layers (depth, normal, albedo, etc.) for use in those.

# Conclusion

Raygon still has a breadth of work before it can be fully usable. Your support will ensure appropriate resources and time can be devoted to ensuring its completion. There is nothing we can't do.

Thank you for reading.

[old_demo]: ./assets/test34.png "Old Demo"
[test173]: ../assets/test173.png "Test 173"
[test218]: ../assets/test218.png "Test 198"
[test216]: ../assets/test216.png "Test 216"
[test213]: ./assets/test213.png "Test 213"