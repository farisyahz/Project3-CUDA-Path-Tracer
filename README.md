# Prismatic Orrery — CUDA Path Tracer

University of Pennsylvania, CIS 565: GPU Programming and Architecture, Project 3

The renderer follows one camera path per pixel and iteration, accumulates linear radiance, and saves both a tone-mapped PNG and a linear HDR image. `scenes/prismatic_orrery.json` stages a glass orb between a woven metallic ring and a gyroid sculpture, with marble and checker materials under warm and cool area lights. All geometry in this scene is defined in the repository; no downloaded models or third-party rendering code were used.

![Prismatic Orrery rendered at 720 × 540 and 1,600 samples per pixel](prismatic_orrery.2026-09-23_21-11-55z.1600samp.png)

This render was generated on an NVIDIA GeForce RTX 4050 Laptop GPU in headless mode. Add the author name and any remaining hardware details before submission. Feature-by-feature performance figures still need controlled comparison runs; they are intentionally not claimed here without those benchmarks.

## Render

Run the executable with `scenes/prismatic_orrery.json` (or `../scenes/prismatic_orrery.json` from a Visual Studio build directory). The scene defaults to 720 × 540 pixels, 1,600 samples per pixel, and 10 bounces. Press `S` to save at any point; `Esc` saves and exits. Both files include a timestamp and sample count. The PNG is display-ready; the HDR retains linear values for later editing.

On laptops where the OpenGL window and CUDA use different GPUs, run `cis565_path_tracer scenes/prismatic_orrery.json --headless` instead. This renders without an OpenGL window and saves PNG/HDR automatically after the scene's iteration count. `scenes/prismatic_orrery_preview.json` is a 640 × 480, 128-sample version for a quicker first image.

The analytics window shows the traced depth and GPU milliseconds per iteration. Its checkboxes toggle material sorting, direct area lighting, and Russian roulette. Change `APERTURE` to `0.0` in a scene to compare the pinhole and thin-lens camera. `FOCUS_DISTANCE` is measured along the camera's forward axis in scene units.

## Implemented features

| Feature | Behavior | Assignment points |
| --- | --- | ---: |
| Diffuse path tracing, material sorting, stochastic antialiasing | Required Part 1 features; paths and intersections sort together by material type | Core |
| Dielectric refraction | Schlick Fresnel choice between reflection and Snell transmission, with total internal reflection | 2 |
| Thin-lens depth of field | Samples a disk aperture and aims rays at a shared focal plane | 2 |
| Procedural shapes and textures | Ray-marched gyroid and three-lobed woven ring; checker and marble material patterns | 4 |
| Direct area lighting | Samples cube and sphere emitters, checks visibility, and avoids duplicate diffuse light hits | 2 |
| Russian roulette | Terminates low-throughput paths after three bounces and divides survivors by their survival probability | 1 |

The selected Part 2 features total **11 points**. A basic torus is also supported for compositions but is not counted as one of the two complex procedural shapes.

## How it works

Each iteration starts with a jittered primary ray. For each bounce, the GPU finds the closest scene intersection. When sorting is on, a key generated from the hit material's behavior groups misses, emitters, glass, mirrors, and diffuse paths. Thrust sorts the path and intersection buffers as a single zipped value so the hit data continues to match its path. The shading kernel adds visible light samples at diffuse surfaces, samples the BSDF for the next direction, and marks finished paths. Those paths contribute once to the image and are compacted away before the next bounce. Random seeds use the original pixel index, so sorting does not change the per-pixel sampling sequence merely by moving a path in memory.

The procedural objects are evaluated in object space inside a bounding sphere. A conservative ray march finds the surface, and finite differences estimate its normal before the object's inverse-transpose transforms that normal to world space. The gyroid is a clipped periodic shell; the woven ring varies its radial and vertical center three times around the circle. Checker texture selects one of two colors by object-space cell parity. Marble texture blends two colors along warped sinusoidal bands. These patterns are applied to the material's diffuse color and therefore affect both indirect and directly sampled light.

Direct lighting samples one emitter uniformly, then a point on its surface in area measure. It multiplies the diffuse BRDF by the geometric cosine and inverse-square terms, divides by the sampling PDF, and traces a shadow ray. Diffuse paths that subsequently hit a sampled emitter do not add the same path's emission a second time; primary and specular paths still see the emitter. The current implementation samples one light point per diffuse bounce and supports transformed cubes and spheres as emitters.

## Scene controls and comparisons

The following comparisons should use the same camera, resolution, sample count, and GPU. Record the analytics window's GPU milliseconds per iteration after the render has warmed up, then save each output. Before/after images and measurements are left for the local render so the README does not imply that unrun experiments were performed.

| Comparison | Change one setting | Expected visual and performance effect |
| --- | --- | --- |
| Material sorting | Toggle `Sort paths by material` | Similar converged image; sorting costs a pass but reduces mixed BSDF work within warps. The net time depends on scene material diversity. |
| Direct lighting | Toggle `Direct area lighting` | Lower variance near finite lights when enabled, with an extra shadow ray at diffuse hits. |
| Russian roulette | Toggle `Russian roulette` | Similar converged image; fewer deep rays in closed scenes, with some added variance. |
| Refraction | Change the glass orb's `TYPE` from `Glass` to `Diffuse` | Refracted background and internal reflections appear only with glass; glass paths may require more bounces. |
| Depth of field | Set `APERTURE` to `0.0` | All depths are sharp with a pinhole; a finite aperture blurs objects away from the focal plane. |
| Procedural geometry | Change `gyroid` and `woven_ring` to `sphere` | Detailed silhouette and internal structure disappear; ray marching is more expensive than analytic sphere intersection. |
| Procedural textures | Remove `TEXTURE`, `RGB2`, and `TEXTURE_SCALE` | Bands and tiles become solid colors; the analytic texture cost is small relative to ray tracing. |

For a hypothetical CPU implementation, the independent pixel paths and shadow rays would parallelize across CPU threads, but the GPU can run far more of them concurrently. Sorting and compacting reduce wasted GPU work and coherence losses, while they introduce memory movement that may be less useful for a CPU's smaller set of wider caches. Ray marching and shadow rays are the primary costs in the showcase scene. Potential improvements include a spatial hierarchy for scene traversal, dedicated shade kernels per material class, light selection weighted by power, multiple importance sampling, and a denoiser. These are future optimization ideas, not implemented features.

## Scene format additions

- Materials accept `TYPE: "Glass"` with `IOR`, or `TYPE: "Specular"` for ideal mirrors. Existing `Diffuse` and `Emitting` types remain supported.
- Diffuse materials accept `TEXTURE: "Checker"` or `"Marble"`, `RGB2`, and `TEXTURE_SCALE`.
- Objects accept `TYPE: "torus"`, `"woven_ring"`, or `"gyroid"` in addition to `cube` and `sphere`.
- Cameras accept `APERTURE` (lens radius) and `FOCUS_DISTANCE`. Omitted aperture defaults to a pinhole camera.

The renderer uses the repository's existing GLM, Thrust, ImGui, GLFW, GLEW, and stb dependencies. The rendering algorithms and showcase scene added here are original to this project.

## References

The implementation follows the rendering concepts in [PBRT's diffuse reflection](https://pbr-book.org/4ed/Reflection_Models/Diffuse_Reflection), [dielectric BSDF](https://pbr-book.org/4ed/Reflection_Models/Dielectric_BSDF), [thin-lens camera](https://www.pbr-book.org/4ed/Cameras_and_Film/Projective_Camera_Models#TheThinLensModelandDepthofField), and [area-light path tracing](https://pbr-book.org/4ed/Light_Transport_I_Surface_Reflection/A_Better_Path_Tracer). The project already includes the other library dependencies.

