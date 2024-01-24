# Chronicles of Crafting a Hybrid RayTracer in DXR



This project was my first experience with 3D ray tracing (and DXR), and I intend to present the features I managed to implement and maybe go through some of the mistakes I realized I made.



![shadow](/assets/All.png) 

# My initial planning for this project:

#### Below is the hierarchy of these features, reflecting my initial strategy and focus areas:

* **Hybrid Pipeline**

* **Shadows**

* **Reflections** 

* Refractions

* **Ambient Occlusion**

* Support for multiple lights

#### In the limited time I had, I managed to implement the bolded features above.

# Some basic theory

## Hybrid Ray Tracing

A hybrid ray tracing pipeline consists of a rasterization pass that writes a G-Buffer that is then used to reconstruct the scene in the ray tracer.

### Using a rasterizer to write a G-Buffer

A G-Buffer is a series of multiple textures that contain data about the geometry within the scene, as it is seen from the perspective of the camera. So basically, what this pass does is use a rasterizer to render multiple render targets that contain screen space information about the scene.

![Hybrid](/assets/Hybrid_RTs.png) 

For my project, I have used 4 render targets, for storing the world position of the pixels, the surface normals, the albedo color and the material information of the surfaces(roughness and metallic, more on this later)


### Why do this?

Rasterizers are very fast, so using the G-Buffer to avoid shooting the primary rays with the ray tracer would give the application a boost in performance(that's true at least for older hardware that was not optimized specifically for ray tracing).


### Using the G-Buffer in the ray tracer

Normally, a ray has to be traced for each of the pixels on the screen, from the camera to the scene. Each ray has to be checked against the geometry in the scene(which is not that slow if you use acceleration structures), and return values like color or distance(these are called the primary rays). 

All of the data that that is collected with the primary rays, can also be collected from the G-Buffer, but with an lower performance cost.

The textures are accessed within the ray generation shader and the data that is provided within them can be used to trace other types of rays(e.g. shadow rays, reflection rays, ambient occlusion rays).

## What can my ray tracer do?

This is a distributed ray tracer. That means that for each type of effect, I am tracing multiple samples and integrating over their contributions. This would cause noise, due to its sthocastic nature, but more samples means a closer result to a realistic illumination, and we always have the option to use the samples from the past frames and improve the result.

### Shadows

In order to achieve realistic shadows, they have to have a "smooth" falloff. This is because in real life, lights are not infinitely small points, but entire areas, causing the light to fall unevenly around an object's shadow, making it softer towards the edges.

Soft shadows and are achieved by tracing multiple samples towards the light source. The ray has to be directed towards any random point on that light souce. This way, the samples closer to the edge of the shadow have a smaller probability of hiting the object that is casting the shadow. By adding up all the results from the samples, the shadow becomes smoother towards the edges.


![shadow](/assets/Soft_Shadow.png) 
![shadow](/assets/Hard_Shadow.png) 

### Ambient Occlusion

Ambient occlusion helps with defining the environmental occlusion of the ambiental light. This is achieved by tracing rays from a surface in random directions and using the number of hits and the distance to the geometry that was hit by the rays to define how occluded is that surface.

![ao](/assets/AO.png) 

### Reflections and PBR
The reflections are the part of indirect lighting, that is affected by the angle of incidence(between the light ray and the surface normal) and the angle of the camera relative to the reflected ray.

In ray tracing, the light is calculated the opposite way light works in real life. A ray is sent from the camera towards a surface, then the angle between the ray and the surface normal is used to calculate the reflected ray, which would hit other illuminated geometry and return the respective color. 

For a more realistic result, PBR(Phisically Based Rendering) is used. PBR defines the properties of a material using 2 values: roughness and metallic. The most widely used illumination modelusing these values is the  Cook-Torrance microfacet model. This model states that surfaces are made from tiny microfacets that reflect the light perfectly, like a mirror.

 The roughness value describes how scattered are the angles of these microfacets. We can apply this information to ray tracing, by sampling random directions for those microfacets, distributing the angle of these based on a mathematical formula. We then use the angle of the microfacets instead of the surface normal, to calculate the reflected rays. This will cause the ray tracer to sample light contributions from a wider angle and cause the material to look rougher, or from a smaller angle if the roughness is lower.

 The metallic value is relevant when calculating the contribution of each reflection ray.

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

The render targets are bound to the pipeline using OMSetRenderTargets. I have to make use of the position within the descriptor heap of the render targes when creating the CPU descriptor handles.

The states of the resources have to be changed from RTV to UAV after the rasterizer pass and swaped back after the ray tracing pass.


Before the ray tracing pass, the hescriptor heap containing the views for the resources needed shall be bound to the pipeline.


The G-Buffer could also be passed as an SRV in the ray tracing pass, since I am doing no writing in this stage on these resources.

Accessing the G-Buffer in the ray generation shader is very convenient. The texture is declared as an array, and the offset from the first UAV declared in the descriptor heap is uased as an index, to get the desired texture.

The data collected from the texture is then used for shadows, reflections and ambient occlusion.

### Ambient Occlusion

Ambient occlusion was the simplest feature to implement, but it's a feature with a great impact. It requires generating a random point on a hemisphere, and tracing a ray in that direction. 

In case no geometry is hit by the ray, a counter is incremented by 1. When some geometry is hit, the hit shader returns the distance to that hit point. The counter is incremented by the distance divided by the maximum length of the ray. 

The counter is then divided by the number of samples, and the returned result is then multiplied with the direct illumination radiance.


![buas logo](/assets/Logo_BUas.png) 

