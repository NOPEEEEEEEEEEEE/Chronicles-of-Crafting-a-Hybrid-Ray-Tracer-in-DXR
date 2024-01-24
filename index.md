
In an time where advanced hardware overshadows the usage of a hybrid ray tracer, the art of using this rendering method remains a valuable exercise for mastering DX12 or any other graphics API. This approach not only sharpens one's skills in managing multiple render targets but also in implementing a deferred pipeline.

This article serves as a source of essential information on hybrid ray tracing, coupled with insights from my personal journey in applying this theory using DX12. The theoretical aspects are universally applicable, regardless of the graphics API you choose. The second half of this article delves into my journey in developing a hybrid ray tracer with DXR and HLSL.

This project was my first experience with 3D ray tracing (and DXR), so please take all the information with a grain of salt.

Before reading the article, I recommend  watching this video as an introduction to ray tracing. While my article lays down the fundamental concepts, additional visual context can greatly enhance your understanding.

![shadow](/assets/All.png) 

# My initial planning for this project:

#### Below is the hierarchy of these features, reflecting my initial strategy and focus areas:

My focus was on building a ray tracer that would run in real time. For that purpose, I chose to implement the following features:

* **Hybrid Pipeline**

* **Shadows**

* **Reflections** 

* Refractions

* **Ambient Occlusion**

* Support for multiple lights

#### In the limited time I had, I managed to implement the bolded features above.

# Some basic theory

## Hybrid Ray Tracing

A hybrid ray tracing pipeline involves a rasterization pass, creating a G-Buffer for scene reconstruction in the ray tracer.

### Using a rasterizer to write a G-Buffer

A G-Buffer comprises multiple textures that hold data about the scene’s geometry from the camera's viewpoint. This step uses a rasterizer to render multiple render targets containing screen space information about the scene.

![Hybrid](/assets/Hybrid_RTs.png) 

For my project, I used four render targets to store the world positions of pixels, surface normals, albedo color, and material information (roughness and metallic properties).


### Why do this?

Rasterizers are fast, so leveraging a G-Buffer to avoid the need for primary rays(the rays traced from the camera towards the scene) in the ray tracer boosts performance, particularly on older hardware not optimized for ray tracing.


### Using the G-Buffer in the ray tracer

Normally, a ray has to be traced for each of the pixels on the screen, from the camera to the scene. Each ray has to be checked against the geometry in the scene and return values like color or distance. 

All of the data that that is collected with the primary rays, can also be collected from the G-Buffer with a lower performance cost.

Textures in the ray generation shader access this data, aiding in tracing other ray types like shadow, reflection, and ambient occlusion rays.

## Capabilities of My Ray Tracer

This distributed ray tracer traces multiple samples for each effect and integrates their contributions.  While this process introduces noise due to its sthocastic nature, more samples yield results closer to realistic illumination. Past frame samples can also be used for refinement.

### Shadows

Realistic shadows require a smooth falloff, as lights in reality are not point sources but areas, causing the light to fall unevenly around an object's shadow, making it softer towards the edges.

Soft shadows and are achieved by tracing multiple samples towards the light source. Each ray targets a random point on the light source, making edge-near samples less likely to intersect the shadow-casting object.

By summing up all the sampled results, the shadow becomes smoother towards the edges.

![shadow](/assets/Soft_Shadow.png) 
![shadow](/assets/Hard_Shadow.png) 

### Ambient Occlusion

Ambient occlusion helps with defining the environmental occlusion of the ambiental light. This is achieved by tracing rays in random directions from a surface and using the number of hits and the distances to impacted geometry to determine surface occlusion.

![ao](/assets/AO.png) 

### Reflections and PBR
The reflections are the part of indirect lighting, that are affected by the angle of incidence(between the light ray and the surface normal) and the angle of the camera relative to the reflected ray.

In ray tracing, light calculation is reversed compared to real life. A ray sent from the camera to a surface uses the angle with the surface normal to compute the reflected ray, which then interacts with other illuminated geometry.

PBR (Physically Based Rendering) enhances realism. It defines material properties with two values: roughness and metallic. The Cook-Torrance microfacet model, a widely used illumination model, considers surfaces as collections of tiny, perfectly reflecting microfacets.

 Roughness affects the scattering of these microfacet angles.In ray tracing, this translates to sampling random microfacet directions, with the distribution based on a mathematical formula. We then use the angle of the microfacets instead of the surface normal, to calculate the reflected rays.  Different angles lead to varied light contributions, altering the material's appearance.

The metallic value influences each reflection ray's contribution.

![reflections](/assets/Reflections.png) 

# My journey implementing all this in DXR

### Setting up DXR
As a code base for this project, I used my Dx12 rasterizer project that I made for my provious uni block. I started by setting up the DXR pipeline, setting up the shader tables, the resource descriptor heap and the acceleration structures.
I used the DXR helpers from NVidia to do so, more resources on this [here](https://developer.nvidia.com/rtx/raytracing/dxr/dx12-raytracing-tutorial-part-1).

The only information I would have to add to the NVidia tutorials would be that I had some allignment issues with their helpers, which required going into "ShaderBindingTableGenerator.cpp" and manually changing the miss shader entry size so that it would allign to 64 bytes.

### Shadows and Random numbers on GPU

Once I reached this stage, I it would be best not to respect my initial plan, and implement the shadows first so that I would have what to test the hybrid pipeline on.

To implement this, I use a function to generate a random point inside a unit sphere, I multiply the resulting vector by the size of the light, and then I add it to the light position. 
I then use the resulting position to calculate the direction and length of the ray from the shaded surface to the random point.

I made sure to avoid calling a hit shader since it would be unnecessary(free performance!). I use 2 flags for the TraceRay function , to avoid unnecessary checks. Only the miss shader is called, returining a value of false.

In case the miss shader is called, i incremented a counter for the hits. After iterating through all the samples, the counter is divided by the number of samples, returning the value of the shadow in that point on the surface. This value is then multiplied with the direct illumination radiance.

In my project, I only have support for one single point light, which makes things much simpler. Here's some directions for [multiple light support](https://blog.traverseresearch.nl/fast-cdf-generation-on-the-gpu-for-light-picking-5c50b97c552b).

When it comes to generating random numbers, this was my first time trying to do it, and I had the unpleasant surprise that there's no function hlsl that can do that for you. Based on my research, the best function to do this would be a PCG HASH. It has both great distribution and is fast compared to other methods. 

This function returns pseudo-random numbers, and we have to provide a seed that is different for each pixel/ sample in order to get random values. There are a bunch of things we could use when building the seed, such as the launch index of the ray(both x and y of the launch index should be used, for good distribution), or the world position of the fragment. 

For temporal accumulation, I also had to include the index of the frame in the seed. Also, for each sample, the sample index can be used can be used to generate different values for each of them. It is ideal to multiply all the values used in a seed with big prime numbers to avoid corelation between the values.


### Reflections

For the reflections, I used a GGX microfacet distribution function that would generate the normal of a microfacet inside a cone, with the angle dependent on the roughness of the material. The generated microfacet is then used to reflect the ray from the camera and trace the ray in this direction. 

When tracing the reflection ray, the hit shader returns the color and distance to the hit point. I used the distance to calculate the location of the hitpoint, also using the direction for the ray and its origin. With this location, I can trace some shadow rays again, to make the shadows more accurate. The traced shadows will be multiplied with the contribution of the ray.

For the contribution of the incoming light, I used a Cook-Torrance BRDF, calculating the normal distribution, geometry factor and the fresnel.

For accurate reflections, the probability of choosing that certain direction for the reflection ray has to be calculated. I used probability density function based on the ggx distribution. 

After adding up all the contributions from all the samples, I divided the result by the number of samples. The returned value is added to the direct illumination, forming the final color of the pixel.


### Hybrid pipeline

For the hybrid pipeline, I am using 4 buffers. For each of them I stored one render target descriptor in a hescriptor heap, and one UAV descriptor in the descriptor heap that I use in the ray tracing pass(UAV's are nice because they allow both reading and writing). 
This way,I can use the same resource for both the rasterizer and for the ray tracer, without having to copy anything.

The render targets are bound to the pipeline using OMSetRenderTargets. I have to make use of the position within the descriptor heap of the render targes when creating the CPU descriptor handles. If you're using ImGUI, make sure you use OMSetRenderTargets to bind your back buffers before rendering it.

The states of the resources have to be changed from RTV to UAV after the rasterizer pass and swaped back after the ray tracing pass.


Before the ray tracing pass, the hescriptor heap containing the views for the resources needed shall be bound to the pipeline.


The G-Buffer could also be passed as an SRV in the ray tracing pass, since I am doing no writing in this stage on these resources.

Accessing the G-Buffer in the ray generation shader is very convenient. The texture is declared as an array, and the offset from the first UAV declared in the descriptor heap is uased as an index, to get the desired texture.

The data collected from the texture is then used for shadows, reflections and ambient occlusion.

### Ambient Occlusion

Ambient occlusion was the simplest feature to implement, but it's a feature with a great impact. It requires generating a random point on a hemisphere, and tracing a ray in that direction. 

In case no geometry is hit by the ray, a counter is incremented by 1. When some geometry is hit, the hit shader returns the distance to that hit point. The counter is incremented by the distance divided by the maximum length of the ray. 

The counter is then divided by the number of samples, and the returned result is then multiplied with the direct illumination radiance.


## Conclusions

The information I presented here was minimal, but I hope it was enough to spark your interest. This project was a great source of information for me, and I totally recommend giving it a try. 

Based on my experience, DXR was a great API to use for this purpose. It is a bit of struggle to understand it in the beginning, but once you get the hang of it, it's a realy great tool.

The foundation I laid with this provides further possibilities for interesting features.

For more information, here are some of the resources I used:

<iframe src="https://cdn.knightlab.com/libs/juxtapose/latest/embed/index.html?uid=f45698fe-b9ff-11ee-9ddd-3f41531135b6" width="100%" height="auto" frameborder="0" scrolling="no"></iframe>

<iframe frameborder="0" class="juxtapose" width="100%" height="360" src="https://cdn.knightlab.com/libs/juxtapose/latest/embed/index.html?uid=f45698fe-b9ff-11ee-9ddd-3f41531135b6"></iframe>



![buas logo](/assets/Logo_BUas.png) 

