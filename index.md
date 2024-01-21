# Introduction

![shadow](/assets/All.png) 

# My initial planning for this project:

Hybrid pipeline
Shadows
Reflections 
Refractions
Ambient Occlusion
Support for multiple lights


# Hybrid Ray Tracing

When we think about a hybrid ray tracer, we think about the usage of both a rasterizer and a ray tracer to present images on the screen. But how does that work?

## Using a rasterizer to write a G-Buffer

A G-Buffer is a series of multiple textures that contain data about the geometry within the scene, as it is seen from the perspective of the camera. So basically, what this pass does is use a rasterizer to render multiple render targets that contain screen space information about the scene.

<table>
  <tr>
    <td><img src="/assets/Final_Position_Rt.png" alt="alt text" style="width:100%"></td>
    <td><img src="/assets/Final_Normal_RT.png" alt="alt text" style="width:100%"></td>
  </tr>
  <tr>
    <td><img src="/assets/Final_Albedo_RT.png" alt="alt text" style="width:100%"></td>
    <td><img src="/assets/Final_Material_Rt.png" alt="alt text" style="width:100%"></td>
  </tr>
</table>

For my project, I have used 4 render targets, for storing the world position of the pixels, the surface normals, the albedo color and the material information of the surfaces(roughness and metallic)


### Why do this?

Rasterizers are very fast, so using the G-Buffer to avoid shooting the primary rays with the ray tracer would give the application a boost in performance(that's true at least for older hardware that was not optimized specifically for ray tracing).

### How to do this with DX12 ?


## Using the G-Buffer in the ray tracer

Normally, a ray has to be traced for each of the pixels on the screen, from the camera to the scene. Each ray has to be checked against the geometry in the scene(which is not that slow if you use acceleration structures), and return values like color or distance(these are called the primary rays). 

All of the data that that is collected with the primary rays, can also be collected from the G-Buffer, but with an(allegedly) lower cost.

The textures are accessed within the ray generation shader and the data that is provided within them can be used to trace other types of rays(e.g. shadow rays, reflection rays, ambient occlusion rays).

### How are the textures accessed within the ray tracer in DXR?


# What can my ray tracer do?

This is a distributed ray tracer. That means that for each type of effect, I am tracing multiple samples and integrating over the contributions, so I could achieve more realistic results.

## Shadows

In order to achieve realistic shadows, they have to have a "smooth" falloff. These are called soft shadows, and are achieved by tracing multiple samples towards the light source. The ray has to be directed towards any random point on that light souce. 

In my project, I only have support for one single point light, which makes things much simpler. 

![shadow](/assets/Shadow.png) 

## Ambient Occlusion

Ambient occlusion helps with defining the environmental occlusion of the ambiental light. This is achieved by tracing rays from a point in random directions and if the ray hits any geometry, the distance from the origin to the hit location is used to evaluate how occluded that point is.

![ao](/assets/AO.png) 

## Reflections and PBR

The reflections 

![reflections](/assets/Reflections.png) 





![buas logo](/assets/Logo_BUas.png) 