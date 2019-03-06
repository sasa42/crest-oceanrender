
# The Tech - Overview

Some of the core ideas were described at SIGGRAPH 2017 in the *Advances in Real-Time Rendering* course (course page [link](http://advances.realtimerendering.com/s2017/index.html)). Since this initial publication we have been actively working to extend the feature set, which includes innovations in the following areas.


## Shape

It is well known that ocean shape can be well approximated by summing Gerstner waves. Dozens of these are required to obtain an interesting shape. In previous implementations this has been prohibitively expensive and shape is either generated using an online FFT, or precomputed and baked into textures.

We generate shape from Gerstner waves efficiently at runtime by rendering at multiple scales, and ensure that waves are never over-sampled (inefficient) or under-sampled (bad quality, aliasing). This is highly performant and gives detail close to the viewer, and shape to the horizon. This gives considerable flexibility in shape and opens possibilities such as attenuating waves based on ocean depth around shorelines.

To control the ocean shape, we introduce an intuitive and fun shape authoring interface - an *equalizer* style editor which makes it fast and easy to achieve surface shape. Art direction such as *small choppy waves with longer waves rolling in from a storm at the horizon* is simple to achieve in this framework. We also support empirical ocean spectra from the literature (Phillips, JONSWAP, etc) which can be used directly or as a comparison.

For interactivity we add a dynamic wave simulation on top of the ocean waves. The simulation is computed on a heightfield and then converted into displacements, which trigger foam generation and enable boat wakes to be generated. This is demonstrated in the *boat* and *threeboats* example scenes.

The final shape is asynchronously read back to the CPU for gameplay/physics use. This gives access to the full, rich shape without requiring expensive CPU calculations or pipeline stalls.


## Mesh

We implement a 100% pop-free meshing solution, which follows the same unified multi-scale structure/layout as the shape data.
The vertex densities and locations can be configured to match the shape texels 1:1, or the geometry can be generated at lower resolution for higher performance.
When running at 1:1 resolution the shape is never over-sampled or under-sampled, giving the same guarantees as described above.

Our meshing approach requires only simple shader instructions in a vertex shader, and does not rely on tessellation or compute shaders or any other advanced shader model features. The geometry is composed of tiles which have strictly no overlap, and support frustum culling. These tiles are generated quickly on startup.

The multi-resolution representation (shape textures and geometry) is scaled horizontally when the camera changes altitude to ensure appropriate level of detail and good visual range for all viewpoints. To further extend the surface to the horizon we also add a strip of triangles at the mesh boundary.


## Shading / VFX

Normal maps are elegantly incorporated into our multi-scale framework. Normal map textures are treated as a slightly different form of shape that is too detailed to be efficiently sampled by geometry, and are sampled at a scale just below the shape textures. This combats typical normal map sampling issues such as lack of detail in the foreground, or a glassy flat appearance in the background.

The ocean colour comes from a subsurface scattering approximation based on view and normal vectors, and primary light direction.

Foam is shaded in two layers. Underwater bubbles have parallax and refraction offsets to give an impression of depth. White foam on top of the water is shaded with a simple 3D lighting model using procedurally generated normals.

Transparency and refraction are also supported, and Schlick's fresnel approximation selects between the refracted colour and a reflected colour. There is an option on the material to boost specular highlights by adding directional lighting if needed.

All shading features are on static switches and can be disabled if not required.

As an area of future work, the branch *fx_test* explores dynamically generating spray particle effects by randomly sampling points on the surface to detect wave peaks (issue #93).


# High level flow

On startup, the *OceanBuilder* script creates the ocean geometry as a LODs, each composed of geometry tiles and a shape camera to render the displacement texture for that LOD.

At run-time, the viewpoint is moved first, and then the *Ocean* object is placed at sea level under the viewer. A horizontal scale is computed for the ocean based on the viewer height, as well as a *_viewerAltitudeLevelAlpha* that captures where the camera is between the current scale and the next scale (x2), and allows a smooth transition between scales to be achieved using the two mechanisms described in the SIGGRAPH course.

Once the ocean has been placed, the ocean surface shape is generated by rendering Gerstner wave components into the shape LODs. These are visualised on screen if the *Show shape data* debug option is enabled. Each wave component is rendered into the shape LOD that is appropriate for the wavelength, to prevent over- or under- sampling and maximize efficiency. A final pass combines the results down the shape LODs (from largest to most-detailed), disable the *Shape combine pass* debug option to see the shape contents before this pass.

The ocean geometry is rendered with the Ocean shader. The vertex shader snaps the verts to grid positions to make them stable. It then computes a *lodAlpha* which starts at 0 for the inside of the LOD and becomes 1 at the outer edge. It is computed from taxicab distance as noted in the course. This value is used to drive the vertex layout transition, to enable a seemless match between the two. The vertex shader then samples the current LOD shape texture and the next shape texture and uses *lodAlpha* to interpolate them for a smooth transition across displacement textures. A foam value is also computed using the determinant of the Jacobian of the displacement texture. Finally, it passes the LOD geometry scale and *lodAlpha* to the pixel shader.

The ocean pixel shader samples normal maps at 2 different scales, both proportional to the current and next LOD scales, and then interpolates the result using *lodAlpha* for a smooth transition. Two layers of foam are added based on different thresholds of the foam value, with black point fading used to blend them.

# LOD data

The backbone of Crest is an efficient LOD'd representation for data that drives the rendering, such as surface shape/displacements, foam values, shadowing data, water depth, and others. This data is stored in a multi-resolution format, namely cascaded textures that are centered at the viewer. This data is generated and then sampled when the ocean surface geometry is rendered. This is all done on the GPU using a command buffer constructed each frame by *BuildCommandBuffer*.

Let's study one of the LOD data types in more detail. The surface shape is generated by the Animated Waves LOD Data, which maintains a set of *displacement textures* which describe the surface shape. A top down view of these textures laid out in the world looks as follows:

![CascadedShapeOverlapped](https://raw.githubusercontent.com/huwb/crest-oceanrender/master/img/doc/CascadedShapeOverlapped.png)

Each LOD is the same resolution (256x256 here), configured on the *OceanRenderer* script.
In this example the largest LOD covers a large area (4km squared), and the most detail LOD provides plenty of resolution close to the viewer.
These textures are visualised in the Debug GUI on the right hand side of the screen:

![DebugShapeVis](https://raw.githubusercontent.com/huwb/crest-oceanrender/master/img/doc/DebugShapeVis.png)

In the above screenshot the foam data is also visualised (red textures), and the scale of each LOD is clearly visible by looking at the data contained within. In the rendering each LOD is given a false colour which shows how the LODs are arranged around the viewer and how they are scaled. Notice also the smooth blend between LODs - LOD data is always interpolated using this blend factor so that there are never pops are hard edges between different resolutions.

In this example the LODs cover a large area in the world with a very modest amount of data. To put this in perspective, the entire LOD chain in this case could be packed into a small texel area:

![ShapePacked](https://raw.githubusercontent.com/huwb/crest-oceanrender/master/img/doc/ShapePacked.png)

A final feature of the LOD system is that the LODs change scale with the viewpoint. From an elevated perspective, horizontal range is more important than fine wave details, and the opposite is true when near the surface. The *OceanRenderer* has min and max scale settings to set limits on this dynamic range.

When rendering the ocean, the various LOD data are sample for each vert and the vert is displaced. This means that the data is carried with the waves away from its rest position. For some data like wave foam this is fine and desirable. For other data such as the depth to the ocean floor, this is not a quantity that should move around with the waves and this can currently cause issues, such as shallow water appearing to move with the waves as in issue 96.

The following sections describe the LOD data types in more detail.


## Animated Waves

The Gerstner waves are split by octave - each Gerstner wave component is only rendered once into the most suitable LOD (i.e. a long wavelength will render only into one of the large LODs), and then a combine pass is done to copy results from the resolution LODs down to the high resolution ones.

Crest supports rendering any shape into these textures. To add some shape, add some geometry into the world which when rendered from a top down perspective will draw the desired displacements. Then assign the *RegisterAnimWavesInput* script which will tag it for rendering into the shappe.

There is an example in the *boat.unity* scene, gameobject *wp0*, where a smoothstep bump is added to the water shape. This is an efficient way to generate dynamic shape. This renders with additive blend, but other blending modes are possible such as alpha blend, multiplicative blending, and min or max blending, which give powerful control over the shape.

The final shape textures are copied back to the CPU to provide collision information for physics etc, using the *ReadbackLodData* script.

The animated waves sim can be configured by assigning an Animated Waves Sim Settings asset to the OceanRenderer script in your scene (*Create/Crest/Animated Wave Sim Settings*). The waves will be dampened/attenuated in shallow water if a *Sea Floor Depth* LOD data is used (see below). The amount that waves are attenuated is configurable using the *Attenuation In Shallows* setting.


## Dynamic Waves

This LOD data is a multi-resolution dynamic wave simulation, which gives dynamic interaction with the water.

One use case for this is boat wakes. In the *boat.unity* scene, the geometry and shader on the *WaterObjectInteractionSphere0* will render forces into the sim. It has the *RegisterDynWavesInput* script that tags it as input.

After the simulation is advanced, the results are converted into displacements and copied into the displacement textures to affect the final ocean shape. The sim is added on top of the existing Gerstner waves.

Similar to animated waves, user provided contributions can be rendered into this LOD data to create dynamic wave effects. An example can be found in the boat prefab. Each LOD sim runs independently and it is desirable to add interaction forces into all appropriate sims. The *FeedVelocityToExtrude* script takes into account the boat size and counts how many sims are appropriate, and then weights the interaction forces based on this number, so the force is spread evenly to all sims. As noted above, the sim results will be copied into the dynamic waves LODs and then accumulated up the LOD chain to reconstruct a single simulation.

The dynamic waves sim can be configured by assigning a Dynamic Wave Sim Settings asset to the OceanRenderer script in your scene (*Create/Crest/Dynamic Wave Sim Settings*).


## Foam

The Foam LOD Data is simple type of simulation for foam on the surface. Foam is generated by choppy water (specifically when the surface is *pinched*). Each frame, the foam values are reduced to model gradual dissipation of foam over time.

User provided foam contributions can be added similar to the Animated Waves. In this case the *RegisterFoamInput* script should be applied to any inputs. There is no combine pass for foam so this does not have to be taken into consideration - one must simply render 0-1 values for foam as desired. See the *DepositFoamTex* object in the *whirlpool.unity* scene for an example.

The foam sim can be configured by assigning a Foam Sim Settings asset to the OceanRenderer script in your scene (*Create/Crest/Foam Sim Settings*). There are also parameters on the material which control the appearance of the foam.


## Sea Floor Depth

This LOD data provides a sense of water depth. This is useful information for the system; it is used to attenuate large waves in shallow water, to generate foam near shorelines, and to provide shallow water shading. It is calculated by rendering the render geometry in the scene for each LOD from a top down perspective and recording the Y value of the surface.

The following will contribute to ocean depth:

* Objects that have the *RegisterSeaFloorDepthInput* component attached. These objects will render every frame. This is useful for any dynamically moving surfaces that need to generate shoreline foam, etc.
* It is also possible to place world space depth caches. The scene objects will be rendered into this cache once, and the results saved. Once the cache is populated it is then copied into the Sea Floor Depth LOD Data.


## Shadow

To enable shadowing of the ocean surface, data is captured from the shadow maps Unity renders. These shadow maps are always rendered in front of the viewer. The Shadow LOD Data then reads these shadow maps and copies shadow information into its LOD textures.

It stores two channels - one channel is normal shadowing, and the other jitters the lookup and accumulates across many frames to blur and soften the shadow data. The latter channel is used to affect scattered light within the water volume.

The shadow sim can be configured by assigning a Shadow Sim Settings asset to the OceanRenderer script in your scene (*Create/Crest/Shadow Sim Settings*).


# Collision Shape for Physics

There are two options to access the ocean shape on the CPU (from script) in order to compute buoyancy physics or perform camera collision, etc.
These options are configured on the *Animated Waves Sim Settings*, assigned to the OceanRenderer script, using the Collision Source dropdown.
These options are described in the following sections.

The system supports sampling collision at different resolutions.
The query functions have a parameter *Min Spatial Length* which is used to indicate how much detail is desired.
Wavelengths smaller than half of this min spatial length will be excluded from consideration.

Sampling the height of a displacement texture is in general non-trivial.
A displacement can define a concave surface with overhanging elements such as a wave that has begun to break.
At such locations the surface has multiple heights, so we need some mechanism to search for a height.
Luckily there is a powerful tool to do this search known as Fixed Point Iteration (FPI).
For an introduction to FPI and a discussion of this scenario see this GDC talk: [link](http://www.huwbowles.com/fpi-gdc-2016/).
Computing this height is relatively expensive as each search step samples the displacement.
To help reduce cost a height cache can be enabled in the *Animated Waves Sim Settings* which will cache the water height at a 2D position so that any subsequent samples in the same frame will quickly return the height.

## Ocean Displacement Textures GPU

This collision source copies the displacement textures from the GPU to the CPU. It does so asynchronously and the data typically takes 2-3 frames to arrive.
 This is the default collision source and gives the final ocean shape, including any bespoke shape rendering, attenuation from water depth, and any other effects.

It uses memory bandwidth to transfer this data and CPU time to take a copy of it once it arrives, so it is best to limit the number of textures copied.
If you know in advance the limits of the minimum spatial lengths you will be requesting, set these on the *Animated Waves Sim Settings* using the *Min Object Width* and *Max Object Width* fields.

As described above the displacements are arranged as cascaded textures which shift based on the elevation of the viewpoint.
This complicates matters significantly as the requested resolutions may or may not exist at different times.
Call *ICollProvider.CheckAvailability()* at run-time to check for issues and perform validation.

## Gerstner Waves CPU

This collision option is serviced directly by the *GerstnerWavesBatched* component which implements the *ICollProvider* interface, check this interface to see functionality.
This sums over all waves to compute displacements, normals, velocities, etc. In contrast to the displacement textures the horizontal range of this collision source is unlimited.

This avoids some of the complexity of using the displacement textures described above, but comes at a CPU cost.
It also does not include wave attenuation from water depth or any custom rendered shape.
A final limitation is the current system finds the first GerstnerWavesBatched component in the scene which may or may not be the correct one.
The system does not support cross blending of multiple scripts.


# Masking Out Surface

There are times when it is useful to mask out the ocean surface which prevents it drawing on some part of the screen.
The scene *main.unity* in the example content has a rowboat which, without masking, would appear to be full of water.
To prevent water appearing inside the boat, the *WaterMask* gameobject writes depth into the GPU's depth buffer which can occlude any water behind it, and therefore prevent drawing water inside the boat.
The *RegisterMaskInput* component is required to ensure this depth draws early before the ocean surface.


# Update / Execution Order

The ocean system updates its state in *LateUpdate*, after game state update and animation, etc.

*OceanRenderer* updates before other scripts and first calculates a position and scale for the ocean based on the view location, and then calls update on the various LOD data types.

Next the rest of the *LateUpdate* bucket runs. Any view-dependent ocean data that wasn't updated by the *OceanRenderer* updates here, such as the Gerstner waves which taylors the wave data based on the LOD scales.

Finally *BuildCommandBuffer* runs after everything else and constructs a command buffer for the ocean. This is executed early in the frame before the graphics queue starts. See the *BuildCommandBuffer* code for the update logic.


# Floating Origin

*Crest* has support for 'floating origin' functionality, based on code from the Unity community wiki. See the original wiki page for an overview and original code: [link](http://wiki.unity3d.com/index.php/Floating_Origin).

It is tricky to get pop free results for world space texturing. To make it work the following is required:

* Set the floating origin threshold to a power of 2 value such as 4096.
* Set the size/scale of any world space textures to be a smaller power of 2. This way the texture tiles an integral number of times across the threshold, and when the origin moves no change in appearance is noticeable. This includes the following textures:
  * Normals - set the Normal Mapping Scale on the ocean material
  * Foam texture - set the Foam Scale on the ocean material
  * Caustics - also should be a power of 2 scale, if caustics are visible when origin shifts happen 

By default the *FloatingOrigin* script will call *FindObjectsOfType()* for a few different component types, which is a notoriously expensive operation. It is possible to provide custom lists of components to the 'override' fields, either by hand or programmatically, to avoid searching the entire scene(s) for the components. Managing these lists at run-time is left to the user.
