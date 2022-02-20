---
layout: post
title:  "VEX Ray Tracer #4: Blinn-Phong Shading"
categories:
- "vex-ray-tracer"
tags:
- "houdini"
- "ray-tracing"
- "vex"
---

How I've been looking forward to this part. When I wrote this ray tracer for the first time, implementing Blinn-Phong shading was one of my proudest achievements. Not because it's complicated or anything (it's actually pretty straight forward), but because it was the first time I translated a technical write up and pseudo-code into something useful for my purpose. It was also the first time where I could've stopped and have a working product. Nothing too impressive in a time where real-time ray tracing is the cool kid on the block.

But this was my kid, ugly and slow, but mine.

***

At this point, I would like to share two links that really helped me understand how this whole thing works and how to implement it.

The first links to a YouTube video by Jeffrey Chastine about [Lighting in OpenGL][lighting-in-opengl] where he goes over general lighting theory, lambertian and blinn-phong reflectance.

The second one links to the [Blinn-Phong reflection model][blinn-phong-reflection-model] Wikipedia page. There are a couple of code snippets for various applications and as usual great images to show whats going on.

***

We are going to start with some housekeeping by adding some controls to our scene objects to change "materials" on the fly and adding a light source, also with some properties. Not the most exciting thing in the world but it make the grand reveal more rewarding when we move the light around or change the objects shininess.

![]({{site.baseurl}}/assets/img/vex-ray-tracer/04.001.png)

As you can see, I moved the scene setup into my ray tracing setup. I don't want to jump back and forth so much to add and remove objects, adjust the shader etc. Especially once we have many objects with many different materials.

```c
vector diffuse_color = chv("diffuse_color");
int use_point_color = chi("use_point_color");
if ( use_point_color ) {
    diffuse_color *= v@Cd;
}
v@Cd = diffuse_color;
v@spec_color = chv("spec_color");
@shininess = ch("shininess");
```

The code is pretty simple. We handpick the color for the diffuse and specular components, adjust the shininess to taste and even have a little tick box multiplying our Cd with the selected diffuse color (the ever-present {0.2, 0.2, 0.2} is obviously the default, hence the dark teapots).

I'm storing all these values in point attributes. The diffuse color will be stored in Cd, so we can keep the viewport preview.

***

Next: a light source. For that we just create a single point in space with two attributes. A color and a strength value. Those two with the distance to the object will be enough to calculate the light.

![]({{site.baseurl}}/assets/img/vex-ray-tracer/04.002.png)

Pretty simple stuff. I used a sphere primitive instead of a point, since they are basically the same thing for what we're using it for and it's easier to see in my screenshot. The sphere is followed by two lines of code setting the light color and intensity (or power).

```c
@Cd = chv("light_color");
@light_power = ch("light_power");
```

I'm once again storing the light color in the Cd for viewport preview.

Nice.

***

And that's all we need to get started. Now let's get into the theory of it all (yey!).

We basically calculating two components. The objects lambertian reflectance and it's specular reflectance.

You might have seen a Lambert shader inside some 3D packages. It basically describes a rough shader with no highlights. It blends between the objects diffuse color at the brightest point and shadow-y darkness (usually black) at the darkest point .

The specular component adds on top of the lambertian. It's basically a sharp highlight on the object which makes the object look more shiny. The color of said highlight and how "sharp" the highlight is, can be defined on the shader.

This will make a lot more sense once we build it, so let's do just that.

***

Let's first gather all the shading and light info we just created.

```c
vector light_pos = point(3, "P", 0);
vector light_color = point(3, "Cd", 0);
float light_power = point(3, "light_power", 0);
    
vector diffuse_color = primuv(1, "Cd", hit_prim, hit_uv);
vector spec_color = primuv(1, "spec_color", hit_prim, hit_uv);
float shininess = primuv(1, "shininess", hit_prim, hit_uv);
```

We're reading in the light properties with the point function since we don't have to interpolate any values in between points like we might have to in the future for objects.

The shading properties are read with the primuv like in [Part 2](https://lbreede.github.io/vex-ray-tracer/2021/01/17/plane-projection.html).

```c
vector hit_nml = primuv(1, "N", hit_prim, hit_uv);
```

We can also get the normal of the this way, which we will need in the next section.

![]({{site.baseurl}}/assets/img/vex-ray-tracer/04.004.png)

***

To calculate both the lambertian and the specular component, we need to calculate four normalized vectors with the data we got. We need:

1. The view direction (V) which is the direction of the hit position to the eye (or pixel), which is basically the negated dir vector we use to shoot the rays into the scene.

2. The object's normal (N) that we can gather the same way as the color using the primuv function.

3. The light direction (L), which is the direction from the hit position to the lights source.

4. The half direction (H), which is the half vector (in between) the view direction and the light direction.

![]({{site.baseurl}}/assets/img/vex-ray-tracer/04.003.png)

Ignore the reflection vector (R). It's being used for the classic Phong shading method.

Extra credit for whomever builds a toggle to switch between both models.

***

Let's start with the lambertian. It's basically the dot product of the light direction and the surface normal. So let's build these first.

```c
vector normal = normalize(hit_nml);
vector light_dir = light_pos - hit_pos;
```

I normalize the normal here in case the hit normal comes in the wrong scale. I also renamed the hit_p variable to hit_pos to enforce a naming convention (light_pos, hit_pos, etc.).

The reason why I don't normalize the light direction just yet, is because we need the distance of the light for later to calculate it's intensity. So before normalizing, I store the length of the light direction vector in a float attribute.

```c
float distance = length(light_dir);
distance = distance * distance;
light_dir = normalize(light_dir);
```

We also need to square the distance since light has a quadratic falloff. The reason why we don't use the power function pow() is because it is significantly slower than just multiplying two values together. So whenever you have an integer as an exponent, multiply the values.

That's all we need for the lambertian.

```c
float lambertian = max(dot(light_dir, normal), 0.0);
```

Since a dot product returns values between -1 and 1, we use the max function as a lower end clamp. Now the value is somewhere between 0 and 1.

And at last, the final color.

```c
color = diffuse_color * lambertian * light_color * light_power / distance;
```

![]({{site.baseurl}}/assets/img/vex-ray-tracer/04.005.png)

And that's it. Now we have three perfectly rough teapots, floating through space.

We can now start playing with the colors of the objects and the light or the position of the light. The scene should still be small enough to give you real time feedback.

***

Now the specular.

For optimization, we can already decide, that if the lambertian value is 0, meaning facing away from the light, we don't have to bother calculating the specular. So we can default the specular to 0.0 and open an if statement after that.

```c
float specular = 0.0;

if (lambertian > 0.0)
{
    // do something!
}
```

Let's start by making these other two vectors we needed, the view direction and the half direction.

As stated above, the view direction should be nothing but the inverse of the dir vector we use to shoot the rays into the scene.

```c
vector view_dir = normalize(-dir);
```

To create the half direction, we need to calculate the half vector between the light direction and the view direction. That as easy as adding the two vectors together and the normalizing them.

```c
vector half_dir = normalize(light_dir + view_dir);
```

Done.

Similar to the lambertian formula, we have to calculate another dot product. This time between the light dir and the view dir. That is because the specular angle is supposed to move whenever the camera moves, since it's pseudo reflection of the light source. The lambertian stays the same, regardless of camera position.

```c
float spec_angle = max(dot(half_dir, normal), 0.0);
```

And again, we clamp the lower end of the dot product.

For now, let's write that spec angle into our specular float and add the whole thing to the color output.

```c
specular = spec_angle;
```

We use a similar formula as for the lambertian.

```c
color = diffuse_color * lambertian * light_color * light_power / distance
	+ spec_color * specular * light_color * light_power / distance;
```

![]({{site.baseurl}}/assets/img/vex-ray-tracer/04.006.png)

We are very close. But as you can see, the highlights are really blown out and the reflection is very brought/rough-looking.

That's because we have not yet introduced our shininess value. It serves as exponent of our spec angle. So instead of writing the spec angle to my specular, let's use the power function on it instead.

```c
specular = pow(spec_angle, shininess);
```

The reason why we have to use the pow function here, is because the shininess value can be a float instead of and integer, plus it's encouraged to change the value to taste.

![]({{site.baseurl}}/assets/img/vex-ray-tracer/04.007.png)

This looks more like it. 

And that's it. That's the whole thing. Not to hard right?

***

Important addition, when researching this topic, you might stumble across two additional components besides the lambert and the specular. Ambient and emission. Those are basically two additional colors added on top of the final color formula. One is for filling in these deep black shadows and the other to create a "glow". I don't like the results I get with the ambient since it uniformly lift the material color linearly towards set value. I prefer adding additional weaker lights to fill in those areas. The emit works sometimes if used in the right context. Both components do not take lights into account. If you do choose to add them, here is the final formula:

```c
color = diffuse_color * lambertian * light_color * light_power / distance
	+ spec_color * specular * light_color * light_power / distance
	+ emit_color
	+ ambient_color;
```

***

I encourage everyone now to play with the shading and light properties.

For example, changing the shininess value to 1 is equivalent to setting the specular to the raw spec angle without the power function and see how the specular highlight behaves when increasing it again.

![]({{site.baseurl}}/assets/img/vex-ray-tracer/04.008.png)

Final result with extra test geo.

***

[Get the scene file here.](https://drive.google.com/file/d/1gtIjoQU5MgT2zWK4DiGnU_G9xBHwONy9/view?usp=sharing){:target="_blank"}

[lighting-in-opengl]:           https://www.youtube.com/watch?v=gFZqzVQrw84
[blinn-phong-reflection-model]: https://en.wikipedia.org/wiki/Blinn%E2%80%93Phong_reflection_model