CUDA Path Tracer
================

**University of Pennsylvania, CIS 565: GPU Programming and Architecture, Project 3**

* Eric Micallef
  * https://www.linkedin.com/in/eric-micallef-99291714b/
  
* Tested on: Windows 10, i5, Nvidia GTX1660 (Personal)

## Intro to path tracing

Path tracing is a computer graphics method of three dimensional rendering images. 

Path tracing simulates many effects, such as soft shadows, depth of field, motion blur, sub-surface scattering, textures and indirect lighting.

In order to get high quality images from path tracing, a very large number of rays must be traced to avoid visible noisy artifacts. Because each ray is data dependent of the other rays this makes path tracing a great fit for a GPU.

Below is a typical render of  a scene showing a reflective and diffuse spheres as well as shadows and lighting.

![](img/nice_render.PNG)

## Performance Analysis

![](img/dof_none.PNG)

The below charts are gathered from the scene shown above. In this scene we have 4 small spheres with some reflection but are mostly diffuse. 

![](img/1_loop.png)

The above chart shows the speed up or slow down associated with enabling each algorithm/technique. We see that in general, material sorting adds too much overhead to see any benefits that may be associated. We see caching helps improve performance as we get to make one less call to our function "compute intersections" and we see that stream compaction helps improve performance by quite a bit because we get to remove some dead rays from the scene and do less work each depth calculation. The rest of the features are essentially free as they are not adding many instructions or complexity to the system.

![](img/1_loop_r.png)

![](img/raw_1_loop.png)


The above chart shows the speed up or slow down associated with enabling each algorithm/technique for release mode. We see a slightly different story with this as caching, stream compaction and material sorting all add overhead to the system. This to me was interesting. I am not sure how some of the code changes to tell this part of the picture.

more information below for each algorithm in the algorithm analysis. 

## Algorith Analysis

### Material Sorting

![](img/Material_Sorting.png)

![](img/Material_Sorting_r.png)

The idea behind material sorting is that we sort our rays by material. All rays with Material ID 1 are contiguous with each other in memory. This can help to reduce thread divergence. If all threads in a warp are computing based off a certain material we would expect that they would mostly behave the same way. For example, all threads with a specular material are all going to reflect. 

In my runs there was not any speedup seen from this. I tried several other runs with more materials and objects in the scene to see if my scene was not complex enough but again saw no real advantage.

I suspect if I added texture mapping or more complex materials the algorithms and branch divergence would be more severe and this is when I would see a benefit.

The chart above shows us how long it takes to actually sort our material. So, each run in debug mode we add about 100ms to our computation time but do not see a benefit of greater than 100ms. So, our material sorting overhead does not outweigh any beneifts that we may see from creating contiguous material memory and reducing thread divergence. This is seen in both debug and release mode.

### First Bounce Caching

![](img/Caching.png)

![](img/caching_r.png)

The idea behind first bounce caching is to reduce repetitive computations. Our compute intersections call takes around 250ms and for larger more complex scenes could take longer. Caching allows us to skip the first one.

Every sample, rays are generated from the camera and enter the scene. for the first sample the rays will go to the same spot. Instead of computing this every time we can cache it on our cpu and just reuse it for the next iterations. thereby we get a free intersections computation.

As we allow more bounces or depth to the scene the benefit of this technique will diminish.

The chart above shows us that with a depth of 1 we do not spend any time computing intersections since we just reuse. The time to copy from this information from CPU to GPU was about .25ms compared to computing the intersection which costed us around 250ms.

In release mode we see no benefit from caching on the cpu side. This can mean that the time spent copying data from the cpu to the gpu is equivalent to the time it takes to execute the function "compute_intersections"


### Stream Compaction 

![](img/Stream_Compaction.png)

![](img/Stream_Compaction_r.png)

The idea behind stream compaction is to remove dead rays ( rays that are no longer bouncing ) from the scene. Dead rays are either rays that have hit a light object, have exited the scene or make no contribution to the scene. 

In our naive path tracer our kernel launches a thread per ray. After the first bounce some rays exit and after more bounces more rays will exit. Yet we will still launch a thread for this ray. 

To remedy this we use stream compaction. After every bounce we can remove the rays that are no longer bouncing. Thus, launching less kernels, thus less work to be done.

This technique sees a significant speedup in performance for debug mode but saw too much overhead being created for release mode.

The above chart shows us the time spent doing our sorting we can see almost an exponential decay in time spent as depth increases.

We see this expoenetial decay in both release and debug mode.

### Anti-aliasing

code can be found in pathtrace.cu in "generate ray from camera".

Anti-aliasing is best done on the gpu because every ray from the camera must be slightly manipulated. Therefore since we have many rays and they are all data dependent it makes sense to compute this on a gpu.

The idea behind anti aliasing is to add some jitter or randomness to the camera rays. Instead of having the rays shoot out the same direction every time, we add a slight random offset. This has the input rays start in a different possibly more interesting directions to accumlate color. What we see with the anti aliasing is edges become a bit smoother. This technique is essentially free because all we are doing is adding some randomness to the first rays direction.

Unfortunately if we use anti aliasing we can not use the First Bounce Cache technique. This is because we are manipulating our first rays just a bit. So the first computation will not always be the same.

We could possibly use both techniques if we made the "random-ness" of anti aliasing more deterministic. For example, we adjust the camera jitter based off of our sample number. IF we did this though we would have to have cache more elements meaning more memory useage.


![](img/combo_alias.jpg)

The left image sphere is a render with anti aliasing. You will notice that the edges are much smoother. 

On the right is a render with no aliasing. You will notice the edges are more abrupt and jagged. 

Both of these images are at the same sample point. 

Below is the full image render even from afar the jagged edges are noticeable! ( if your monitor is big enough that is ...)


![](img/cornell_no_alias.png)

No Alias

![](img/cornell_alias.png)
 
Anti Aliasing

![](img/zoom_nice_no_aa.png)

No Alias

![](img/zoom_nice_aa.PNG)

Anti Aliasing

## depth of field

code for implementation can be found in pathtrace.cu function "ConcentricSampleDisk".

Depth of field is best done on the gpu because every ray from the camera must be slightly manipulated based on our lens and focal point. Therefore, since we have many rays and they are all data dependent it makes sense to compute this on a gpu.

From our chart we see depth of field did not add much, if any overhead to the system.
Similar to anti-aliasing this technique is done once per iteration and makes adjustments to the input rays direction and origin. the compute time for this is pretty minimal there by it is masked away by the heavier functions. 

Similar to anti aliasing we can not employ our caching technique speed up when doing depth of field. This is so because we are manipulating the first ray differently each iteration to get that blur effect.


Below is a render of what you would see if you have perfect vision (no depth of field)

![](img/no_blur.PNG)

Below is what you would see if you had decent eye sight or over the age of 60 

![](img/blur.PNG)

And finally, this is what I see without my contacts

![](img/full_blur.png)

## Motion Blur

code for implementation can be found in pathtrace.cu, function "add_blur"

Similar to depth of field and anti aliasing motion blur is pretty much free. The drawbck is that again, we can not use the caching technique. This is so because we moving the object position based off its speed every iteration.

This algorithm has us update the position by a little bit every iteration. So, we just need to loop through however many objects are in the scene and update it based on the speed of the object. Unless there are 1000's of objects in a scene this can and should  be done easily and quickly on the CPU.

Below is an image with alot of smaller spheres flying around the scene. Simulating something like electrons around a nucleus.


![](img/motion_blur_sun.PNG)


## Refraction ( Extra ? )

code for implementation can be found in interactions.h, function "compute_refraction"

refraction is best done on the gpu because every ray is data dependent and the amount of rays to compute per picture is large.

From our results above we see that refracion addded a bit of overhead but not too much. This is expected as adding refraction adds a bit more branch divergence. But the time spent computing refraction is not significantly more than computing a reflective or diffuse object so we only see a slight slow down.

Below is a scene with objects that have some refractivity and a slight bit of reflectivity. We see the sphere in the back refract blue light onto the white wall. We see the yellow sphere obscured from the refractive sphere in front and on the right we see a neat looking cube where we actually see a small reflection inside due to the phenomena of total internal reflection. 


![](img/good_refraction.PNG)


Below is a render with my attempt to simulate refractive water. To do this I just added a thin blue refractive material. It is not the prettiest render but it shows use of fresnels number when combining refraction and reflection.


![](img/water_refraction.PNG)


From the image above we can see the distortion on the walls and slight distortion of the tower.

In real life, water and other refractive materials have some reflection to them. When you look into a lake if it is calm enough you can see a reflection. This is where Fresnels Number can create a more realistic render. 

In the below image we add some reflective and refractive properties to the "water" and use fresnels number to calculate how much reflectivity this ray has. We can see how the middle area has a little glean to it. and the "water" is a different color depending on where it is in the frame. If we could add textures this could become even more realistic!

![](img/water_refraction_p2.PNG)


### Importance Sampling ( Extra )

code for implementation can be found in interactions.h wrapped in #ifdef

Impportance sampling is best done on the gpu because we are adding weight to our ray during diffusion,reflection or refraction. Since we have thousands of rays per image all data dependent the computation is best done on the gpu side.  

From our chart we can see that importance sampling does not add really any overhead to the system ( about 20ms added ). That is because it is just a few extra  multiplies and divides per thread in the shader to get a weight. 

The theory behind importance sampling is that it will help the image converge much quicker. I did not notice a noticeable difference of when the image converges while using importance sampling vs not using importance sampling. At iterations 50, 250, 550 the image roughly looked the same. No worse, no better.

During a regular render we would accumulate color per depth. With importance sampling we want to accumulate color per depth as well but also give some weight to the color of our ray bounces based off of where they hit. 

For example, when we have a diffuse bounce we can bounce anywhere in our cosine weighted hemisphere but some spots in this hemisphere shine brighter than others. With importance sampling we can give more weight to these and less weight to areas that are not as interesting. This helps the image converge quicker.

### Subsurface scattering ( Extra )

Currently a work in progress.


### References

Physically Based Rendering second and third edition
	- Used this books algorithms to employ Depth of field
	- Used book for refraction
	- Used book for fresnels
	- Used book for importance sampling

https://www.scratchapixel.com/lessons/3d-basic-rendering/introduction-to-shading/reflection-refraction-fresnel
	- in help with refraction

https://en.wikipedia.org/wiki/Snell%27s_law
	- more help with refraction and fresnels

https://computergraphics.stackexchange.com/questions/2482/choosing-reflection-or-refraction-in-path-tracing
	- for help with fresnels

https://github.com/ssloy/tinyraytracer/wiki/Part-1:-understandable-raytracing
	- in general helped with the basics of fresnels, reflection, refraction

Ziad and Hannah, Our TA's for help with motion blur

https://learning.oreilly.com/library/view/physically-based-rendering/9780128007099/B9780128006450500130_2.xhtml
	- Help with importance sampling

https://computergraphics.stackexchange.com/questions/4979/what-is-importance-sampling
	- Help with importance sampling

https://www.scratchapixel.com/lessons/3d-basic-rendering/global-illumination-path-tracing/global-illumination-path-tracing-practical-implementation
	- Help with importance sampling
	
	
## Cmakelist

modified defualt for use with tinygfl but did not get it working so regular Cmake file can be used.




