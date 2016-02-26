---
layout: post
title: "Gradient Domain Fusion"
excerpt: "Project 2 for CSE555: Computational Photography."
categories: articles
tags: [comp-photo, image, matlab]
comments: true
share: false
---

<figure>
    <img src="/images/gradient_domain_fusion/sf-npr-big.png" alt="image">
    <figcaption>
        NPR Rendering. Golden Gate Bridge in San Francisco, California.
    </figcaption>
</figure>

Gradient-domain image processing is a technique with numerous applications. This project explores image blending and non-photo realistic rendering. Given an input image, gradient domain image processing first finds the gradient of the given image, and then solves for a new image using a system of equations. The equations act as constraints, defining what problem is solved. In this project, we will explore a simple toy problem, Poisson blending, and non-photo realistic rendering. The code for can be found [here](https://github.com/jmecom/gradient-domain-fusion). Jordan Mecom and I worked together on this project.

## Toy Example

To get started with linear program solvers in MATLAB, we got our feet wet with a little program that solves for the original image. The constraints are given as the x and y directional gradients of the original image and a single pixel intensity as the seeding value. This solver should return the original image, which only has one channel. We evaluated our final image with an error value, which was the sum of square differences between the original image and the resulting image.

The gradients define the change in intensity from a pixel to its neighboring pixels, so with one pixel predefined, there can only be one resulting image. We built the `A` constraint matrix with two constraints per pixel - a constraint for the gradient in the x-direction and one for the y-direction. There was one additional constraint at the end to specify the intensity of one seeding pixel. The `b` matrix contains the actual value of the image gradients and the seeding intensity. The algorithm for toy reconstruction is:
{% highlight python %}
let img be an input

for each pixel (y,x) in img
    # Objective 1
    A(e, y, x+1) = 1;
    A(e, y, x)   = -1;
    b(e) = source image x gradient
    e += 1;

    # Objective 2
    A(e, y+1, x) = 1;
    A(e, y, x)   = -1;
    b(e) = source image y gradient

# Objective 3
A(e, 1, 1) = 1
b(e) = img(1,1)
{% endhighlight %}

Our objective function was to minimize the error value. We simply solved for these constraints using `A\b`. The result is below:

<center>
    <figure>
        <img src="/images/gradient_domain_fusion/toy1.png" alt="image">
        <figcaption>
            Azura from Fire Emblem Fates, which is a great game by the way!
        </figcaption>
    </figure>
</center>

To really understand what was going on, we messed with the various parameters in the linear program. The first image is what happens when we don't use y gradient constraints (error of 151.2625), the second doesn't use x gradient constraints (error of 159.6582), and the last gif shows the image solved with various starting intensities. The intensities start at 1, resulting in the darkest image, to .05, which results in the lightest image. The error values were largest around 1 and .05 and were smallest around .5. This is because the true intensity of the first pixel of the image is `0.4745`.

<center>
    <figure>
        <img src="/images/gradient_domain_fusion/toy_no_y.png" alt="image">
        <img src="/images/gradient_domain_fusion/toy_no_x.png" alt="image">
        <img src="/images/gradient_domain_fusion/toy.gif" alt="image">
    </figure>
</center>

At first, we tried to directly edit the values of the sparse matrix in each loop. Our program ran significantly faster once we built arrays to instantiate the sparse matrix at the very end. The run time was `4.96 seconds` on average.

## Poisson Blending

The idea behind Poisson blending is to take two images - a source and a target - and blend the source onto the target image in a convincing way. The edges of the source image tries to match the color of the target as close as possible while the interior pixels try to preserve the original gradients as close as possible. This is accomplished by solving for the new image using a Poisson equation, defined as follows:

<figure>
    <img src="/images/gradient_domain_fusion/poisson_eq.png" alt="image">
    <figcaption>
        Poisson image editing equation, Perez et al.
    </figcaption>
</figure>

To implement Poisson blending, one can express the problem as a linear program and solve the equation `Av = b` where `v` is the vector to be solved. The vectors `v` and `b` are the size of the pixel count of the background image and `A` is pixel count by pixel count in size.

The algorithm we implemented is shown below, in pseudocode:

{% highlight python %}
let source, target, and mask be inputs

for each channel
  loop over pixels p in target
    if p is outside of the mask,
      # Just keep the background in this case.
      A(e,p) = 1
      b(e) = target(y,x)  # e indexes the current equation.

    else,
      A(e,p) = 4
      b(e) = source gradient at p

      for each neighbor n of p
        if n is inside mask,
          b(e) += target(n) # Handles second sum in Poisson equation
        else,
          A(e,n) = -1         # Handles first sum in Poisson equation
  solve Av = b

combine the channels and return the blended image
{% endhighlight %}

The source gradient is simply the gradient in all four directions. The equation is `source(y,x)*4 + (source(y-1,x) + source(y+1,x) + source(y,x+1) + source(y,x-1))`, if pixel `p` is at `(y,x)`. We set `A(y,x) = 4` because one row of `A` handles each gradient direction. It's possible to split the equations out to one per row, but this was easier. The `A(n) = -1` terms correspond to the `v_i - v_j` term in the first sum of the Poisson equation, which is only set if the `jth` pixel is in the mask. Otherwise, `v_i - t_j` is used, and so we add the target to `b`.

The results of Poisson Blending can be seen below

<figure class="half">
    <a href="/images/gradient_domain_fusion/poisson/arial.jpg" alt="image">
        <img src="/images/gradient_domain_fusion/poisson/arial_preview.png" alt="image">
    </a>
    <a href="/images/gradient_domain_fusion/poisson/ocean.jpg" alt="image">
        <img src="/images/gradient_domain_fusion/poisson/ocean.jpg" alt="image">
    </a>
    <a href="/images/gradient_domain_fusion/poisson/ocean_overlay.png" alt="image">
        <img src="/images/gradient_domain_fusion/poisson/ocean_overlay.png" alt="image">
    </a>
    <a href="/images/gradient_domain_fusion/poisson/ocean.png" alt="image">
        <img src="/images/gradient_domain_fusion/poisson/ocean.png" alt="image">
    </a>
    <figcaption>Clockwise: source, target, Poisson, naive</figcaption>
</figure>

<figure class="half">
    <a href="/images/gradient_domain_fusion/poisson/duck2.jpg" alt="image">
        <img src="/images/gradient_domain_fusion/poisson/duck2_preview.png" alt="image">
    </a>
    <a href="/images/gradient_domain_fusion/poisson/canyon.jpg" alt="image">
        <img src="/images/gradient_domain_fusion/poisson/canyon_preview.png" alt="image">
    </a>
    <a href="/images/gradient_domain_fusion/poisson/canyon_overlay.png" alt="image">
        <img src="/images/gradient_domain_fusion/poisson/canyon_overlay.png" alt="image">
    </a>
    <a href="/images/gradient_domain_fusion/poisson/canyon.png" alt="image">
        <img src="/images/gradient_domain_fusion/poisson/canyon_prewview.png" alt="image">
    </a>
    <figcaption>Clockwise: source, target, Poisson, naive</figcaption>
</figure>

<figure class="half">
    <a href="/images/gradient_domain_fusion/poisson/jar.jpg" alt="image">
        <img src="/images/gradient_domain_fusion/poisson/jar.jpg" alt="image">
    </a>
    <a href="/images/gradient_domain_fusion/poisson/sand.jpg" alt="image">
        <img src="/images/gradient_domain_fusion/poisson/sand.jpg" alt="image">
    </a>
    <a href="/images/gradient_domain_fusion/poisson/sand_overlay.png" alt="image">
        <img src="/images/gradient_domain_fusion/poisson/sand_overlay.png" alt="image">
    </a>
    <a href="/images/gradient_domain_fusion/poisson/sand.png" alt="image">
        <img src="/images/gradient_domain_fusion/poisson/sand.png" alt="image">
    </a>
    <figcaption>Clockwise: source, target, Poisson, naive</figcaption>
</figure>

However, Poisson blending didn't quite always work. Discoloration occurs in the image below of the dog blended with the picture of SR71 Blackbird pilots. The algorithm tends to work best for pictures where the source can pick up colors from the target and still look plausible. For example, pictures of a bear blended into water, a sun into sky, or a rainbow into clouds look good because picking up colors from the target helps the blending process. However, picking up colors when pasting a dog onto a runway doesn't produce as good of an image.

<figure class="half">
    <a href="/images/gradient_domain_fusion/poisson/shibablack.jpg" alt="image">
        <img src="/images/gradient_domain_fusion/poisson/shibablack.jpg" alt="image">
    </a>
    <a href="/images/gradient_domain_fusion/poisson/SR71_crew.jpg" alt="image">
        <img src="/images/gradient_domain_fusion/poisson/SR71_crew.jpg" alt="image">
    </a>
    <a href="/images/gradient_domain_fusion/poisson/blackbird_overlay.png" alt="image">
        <img src="/images/gradient_domain_fusion/poisson/blackbird_overlay.png" alt="image">
    </a>
    <a href="/images/gradient_domain_fusion/poisson/blackbird.png" alt="image">
        <img src="/images/gradient_domain_fusion/poisson/blackbird.png" alt="image">
    </a>
    <figcaption>Clockwise: source, target, Poisson, naive</figcaption>
</figure>

## Mixed Gradients

Mixed gradients is a technique for image blending that is essentially the same as Poisson Blending, except instead of always using the source gradient, the `max(abs(source gradient), abs(target gradient))` is used to select either the source or target gradient. Mixed gradients notably outperform Poisson blending in some cases, as shown below.

<figure class="half">
    <a href="/images/gradient_domain_fusion/washu.jpg" alt="image">
        <img src="/images/gradient_domain_fusion/washu_preview.png" alt="image">
    </a>
    <a href="/images/gradient_domain_fusion/rainbow.jpg" alt="image">
        <img src="/images/gradient_domain_fusion/rainbow_preview.png" alt="image">
    </a>
    <a href="/images/gradient_domain_fusion/washu_mask.png" alt="image">
        <img src="/images/gradient_domain_fusion/washu_mask.png" alt="image">
    </a>
    <a href="/images/gradient_domain_fusion/washu_overlay.png" alt="image">
        <img src="/images/gradient_domain_fusion/washu_overlay.png" alt="image">
    </a>
    <a href="/images/gradient_domain_fusion/washu_mixed.png" alt="image">
        <img src="/images/gradient_domain_fusion/washu_mixed.png" alt="image">
    </a>
    <a href="/images/gradient_domain_fusion/washu_poisson.png" alt="image">
        <img src="/images/gradient_domain_fusion/washu_poisson.png" alt="image">
    </a>
    <figcaption>(Top Left) Original target. (Top Right) Original source. (Mid Left) Mask. (Mid Right) Naive. (Bottom Left) Mixed. (Bottom Right) Poisson. </figcaption>
</figure>

<figure class="half">
    <a href="/images/gradient_domain_fusion/brick-wall.jpg" alt="image">
        <img src="/images/gradient_domain_fusion/brick-wall.jpg" alt="image">
    </a>
    <a href="/images/gradient_domain_fusion/dd_resize.png" alt="image">
        <img src="/images/gradient_domain_fusion/dd_resize.png" alt="image">
    </a>
    <a href="/images/gradient_domain_fusion/text_mask.png" alt="image">
        <img src="/images/gradient_domain_fusion/text_mask.png" alt="image">
    </a>
    <a href="/images/gradient_domain_fusion/text_overlay.png" alt="image">
        <img src="/images/gradient_domain_fusion/text_overlay.png" alt="image">
    </a>
    <a href="/images/gradient_domain_fusion/text_mixed.png" alt="image">
        <img src="/images/gradient_domain_fusion/text_mixed.png" alt="image">
    </a>
    <a href="/images/gradient_domain_fusion/text_poisson.png" alt="image">
        <img src="/images/gradient_domain_fusion/text_poisson.png" alt="image">
    </a>
    <figcaption>(Top Left) Original target. (Top Right) Original source. (Mid Left) Mask. (Mid Right) Naive. (Bottom Left) Mixed. (Bottom Right) Poisson. </figcaption>
</figure>

In both images, the blurriness around the mask is removed or reduced with mixed gradients. In particular, using mixed gradients is a good idea when the source image is partly transparent. The algorithm chooses whichever ''source or destination structure is the more salient at each location'' (Perez, et al.) and achieves a better quality output.

## Extra Credit

We implemented non-photo realistic rendering as extra credit. Our reference for this section was [GradientShop](http://grail.cs.washington.edu/projects/gradientshop/), particularly the slides from a SIGGRAPH 2010 talk, which can be found [here](http://grail.cs.washington.edu/projects/gradientshop/).

This problem was constructed similarly, but instead of solving a Poisson equation, we solved the energy function specified in the linked slides (specifically on slide 74). This function is defined as follows

{% highlight python %}
let u be the unfiltered input image
let f be the filtered output image
let gx, gy be desired pixel-differences
let d be desired pixel-values
let wx, wy, wd be constraint weights

Then the energy function is
min_f { wx(fx - gx)^2 + wy(fy - gy)^2 + wd(f - d)^2 }
{% endhighlight %}

The choice of `gx, gy`, `d`, and the weights all depend on the specific result you're trying to achieve. Our algorithm for non-photo realistic rendering is
{% highlight python %}
# Calculate edges
let ux, uy be x and y gradients of u
let e = sqrt(ux.*ux + uy.*uy)

# Specify everything
gx = e(p) .* ux   # p is a pixel
gy = e(p) .* uy
d = u
wx, wy, wd = 1

# Solve
solve Af = b similarly to above to calculate f

# Stylize
overlay the complement of e onto f to achieve an outline effect
{% endhighlight %}

On average, the algorithm took `19.215 seconds` to run on each image. The results are below.

<figure class="thirds">
    <img src="/images/gradient_domain_fusion/baseballwow.png" alt="image">
    <figcaption>
        (Left) Original Image.  (Middle) Our NPR Blending.  (Right) GradientShop's NPR Blending, Bhat et al.
    </figcaption>
</figure>

<figure class="half">
    <img src="/images/gradient_domain_fusion/flower.png" alt="image">
    <img src="/images/gradient_domain_fusion/flower_npr.png" alt="image">
    <img src="/images/gradient_domain_fusion/sf.png" alt="image">
    <img src="/images/gradient_domain_fusion/sf_npr.png" alt="image">
    <img src="/images/gradient_domain_fusion/rilafood.png" alt="image">
    <img src="/images/gradient_domain_fusion/rilafood_npr.png" alt="image">
    <img src="/images/gradient_domain_fusion/kojima.png" alt="image">
    <img src="/images/gradient_domain_fusion/kojima_npr.png" alt="image">
    <figcaption>
        (Left) Original image.  (Right) NPR blended image.
    </figcaption>
</figure>

In the last image of Hideo Kojima, the lines are relatively thick compares to his facial features, so it obstructs his face and makes him unrecognizable. Faces require a lot more attention to detail and finer lines. If we were to improve our algorithm, we should resize the line width according to the image size. This would fix the issues of harsh lines on the small picture of Kojima.