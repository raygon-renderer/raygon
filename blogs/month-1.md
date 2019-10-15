This Month in Raygon 1
========================

It's been about 50 days since I first announced Raygon, a WIP high performance CPU path tracer written in the Rust programming language, which will eventually feature state of the art light transport integrators including path tracing, bidirectional path tracing and VCM.

In that time I've worked day and night to build what you're about to see. In this post I'll go over some of those things and show some recent renders, then talk about the future of Raygon and how you can help.

## Look back

However, before we get into all of that, I think it's important to show where the project was at on August 24th:

<details>
<summary>
Click to expand the old WIP image
</summary>

![Old Demo][old_demo]
</details>

This image was rendered only a couple weeks after getting triangles to the screen at all, and I was quite excited to share the progress. However, it's not very impressive in retrospect.

At that time, I had spent five or six months working on the foundations for Raygon, including most notably an extremely high performance virtual machine for complex procedural material evaluation, and a low-level linear algebra library from scratch based on `packed_simd`.

# Current Status

Since the first announcement, I've implemented:

* Full path tracing
* Pixel Filter Importance Sampling (FIS)
    * All renders shown below use a Blackman-Harris filter
    * FIS is generally equal or better quality than traditional splatting, while being incredibly easy to implement and integrate into highly concurrent rendering systems like Raygon.
* Physically based materials
    * GGX Microfacet distribution for specular and Oren-Nayar for diffuse
        * About halfway through implementing multiscatter GGX for even more accurate materials
    * Uses the relatively recent GGX Visible-Normal Distribution Function
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

Though, I think the best way to show where the project is at today is visually:

The same living room now:
![Test 198][test198]

Here is an extreme stress-test for object instancing with over **65 Billion** effective triangles, and also using an HDRi with environment importance sampling. Some of the tree leaves also used alpha-masked materials for greater detail.
![Test 216][test216]

Cornell-like Box with the classic Dragon and Buddha
![Test 173][test173]

And here, just a few hours before writing this, I added initial support for physically based metals using conductor Fresnel effects for their color. This one also uses an importance-sampled HDRi for lighting.
![Test 213][test213]

# The Near Future

Before going into what features are planned and in-progress, I want to make it clear that I cannot continue this project without your help. Consider supporting Raygon on [Patreon](https://www.patreon.com/raygon) or [Ko-Fi](https://www.ko-fi.com/raygon). All donations count torwards licenses when Raygon eventually fully releases. Furthermore, donating just $30 will give you a lifetime personal license, and $50 will get your name featured in the executable itself.

Feel free to [Join our Discord Server](https://discord.gg/Y54gQxH) as well.

Now then, with that out of the way, onto the real meat and potatoes.

## Planned Features

These aren't in any particular order.

### Spectral Rendering

One of the biggest features in the works right now is spectral rendering, which will allow for reflective/refractive dispersion, thin-film interference, perfect blackbody radiation,  fluorescence, and polarization. My implementation will be based on the Hero-wavelength method, with some new advancements in color upsampling.

### Bidirectional Path Tracing and SPPM/VCM

With the unidirectional path tracer mostly complete at this point aside from volumes, in the next month or so work will begin on the bidirectional path tracer, and shortly after that Stochastic Progressive Photon Mapping. With those two implemented, VCM (Vertex Connection and Merging) can be built off of that, which combines the best of both in a relatively simple integrator that is remarkably effective.

### Scene Description Format(s)

Currently, scenes are either procedurally generated (such as the grass and trees render) or loaded from `.obj`/`.mtl` files, with materials being generated from the `mtl`.

I'd like to spend time on creating a formal scene description format for anything and everything Raygon can render, in a way that is both human and machine readable/writable.

### Websocket API

Of course, a file format isn't sufficient for interactive rendering from a plugin such as in Blender. To that end, the next step is to create a websocket-based API for communicating assets and rendered images (final and in-progress pixels/tiles) between Raygon and plugins. Perhaps even an extra small web interface for monitoring render progress in realtime.

## Refinement of Existing Features

There is always new research being published, and I want to incorperate the latest graphics science into Raygon.

### Multiscatter GGX

This is a fairly high priority feature that needs to be implemented as soon as possible. It gives much greater realisism to rough surfaces and especially rough glass, as it does not lose energy from ignored scattering events in the material surface.

### Better Samplers and Samples

Currently, Raygon is just using a plain old dumb random sampler for everything. Random samples are about as bad as it gets.

This was chosen originally because for one it's incredibly easy to implement and use, and second because traditional stratified samplers were not initially possible given some of the other features I've implemented (i.e. material shaders can sample random numbers, adaptive number of samplers per pixel, etc.).

However, very recently a new algorithm has come to my attention for *progressively* generating stratified samples, which is ideal for Raygon. I plan to implement this very soon, and doing so should reduce noise in renders drastically.

### Textures

Although basic textures are implemented, the next step is to implement MIP mapping and anisotropic trilinear filtering. Additionally, normal maps should ideally be compressed to use only two texture channels (as they are unit vectors, it's very easy to recalculate a missing component) to save space.

Long-term goals with textures are to implement a megatexture-like streaming system for transparently loading and unloading parts of textures as they are used, which can drastically reduce core memory usage.

### Deformation with Motion Blur

Vertex displacement and mesh deformation is planned, with support for ray-traced motion blur. Animated fluids, morphs and skeletal meshes should be made possible by this.

# Conclusion

Raygon is the culmination of my years of self-taught knowledge and desires for the best renderer possible. While it's not there yet, I'm determined to see it through. With your support and donations, time is the only factor in what can be accomplished.

[old_demo]: ./assets/test34.png "Old Demo"
[test173]: ../assets/test173.png "Test 173"
[test198]: ../assets/test198.png "Test 198"
[test216]: ../assets/test216.png "Test 216"
[test213]: ./assets/test213.png "Test 213"