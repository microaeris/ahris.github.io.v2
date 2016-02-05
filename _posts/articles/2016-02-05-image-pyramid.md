---
layout: post
title: "Image Alignment with Pyramids"
excerpt: "Project 1 for CSE555: Computational Photography."
categories: articles
tags: [comp-photo, images, python]
comments: true
share: false
---

<figure>
    <a href="/images/image_pyramids/00345u_color_2.png" alt="image"><img src="/images/image_pyramids/00345u_color_2_preview.png" alt="image"></a>
    <figcaption>Photo credits to the Library of Congress and Sergey Prokudin-Gorskii.</figcaption>
</figure>

In this project, the goal was to read in the [glass plate images](http://www.loc.gov/pictures/search/?q=Prokudin+negative&sp=2&st=grid) by [Sergey Prokudin-Gorskii](https://en.wikipedia.org/wiki/Sergey_Prokudin-Gorsky) and align the three color channels into one full-color, composite photo. The first part of the project implements alignment using a single-scale algorithm for smaller images. The second part uses an image-pyramid to align high-resolution pictures. To improve the visual quality of the final images, I also cropped the images to remove the excess borders. The code can be found [here](https://github.com/Ahris/Pyramid).

## Approach and Algorithm

Here is a general overview and pseudocode for my project. It can be roughly broken down into the follow steps:

{% highlight html %}
1. Read in the image and trim the uniform white border
2. Crop the image into thirds
3. Remove the jagged black border (more details in the Extra Credits section)
4. Call the single or multi-scale alignment function
5. Combine the RBG color channels in a 3D array
6. Save the image
{% endhighlight %}

### Single-scale Alignment

`Single-scale alignment` is relatively straight forward. The green and red channels are aligned to the blue channel, which acts as the anchor. Displacement vectors for the green and red channels are returned from the function along with the aligned red and green channels. We assume the offset is a translation in the x or y direction.

To find the offset, we first run the image through a Sobel edge filter to remove the noise and to only match the main features. Within a given range, every (x,y) offset combination is tried and evaluated for how well the shifted channels match the blue channel. The channels are evaluated using a sum of squared differences: `sum(pow(image1 - image2, 2))`

The displacement with the lowest score (which was usually around 11,000,00) is then returned. Over the average of 10 runs, this algorithm takes `4.53171 seconds`. And now for the fun part. Here are some of the resulting images.

<figure class="half">
    <img src="/images/image_pyramids/00403v_color.jpg" alt="image">
    <img src="/images/image_pyramids/00402v_color.jpg" alt="image">
    <img src="/images/image_pyramids/00345v_color.jpg" alt="image">
    <img src="/images/image_pyramids/01531v_color.jpg" alt="image">
    <img src="/images/image_pyramids/00212v_color.jpg" alt="image">
    <img src="/images/image_pyramids/00540v_color.jpg" alt="image">
    <figcaption>Single-scale aligned glass plate negatives.</figcaption>
</figure>

### Multi-scale Alignment

`Multi-scale alignment` follows a similar vein. The main difference is that the photos are shrunk to a manageable size and then evaluated. Once the smallest image's displacement is found, it is scaled up and re-checked as we traverse back up the pyramid to the original, large image. I implemented this function recursively:

{% highlight python %}
if image size < 150px:
    Single-scale align the red and green channels
else:
    Gaussian blur the image
    Resize the image to half its original size
    offset = multi_scale_align() * 2 # scale up the recursive result
    Return updated offset from single_scale_align(offset)
{% endhighlight %}

Since multi-scale alignment calls the single-scale process, the evaluation function is the same. The run time of this algorithm was `12.653 seconds` over the average of 10 runs. Now, here are some more photos for your enjoyment.

<figure>
    <a href="/images/image_pyramids/factory_color.png" alt="image"><img src="/images/image_pyramids/factory_color_preview.png" alt="image"></a>
    <a href="/images/image_pyramids/church_color.png" alt="image"><img src="/images/image_pyramids/church_color_preview.png" alt="image"></a>
    <a href="/images/image_pyramids/guy_color.png" alt="image"><img src="/images/image_pyramids/guy_color_preview.png" alt="image"></a>
    <a href="/images/image_pyramids/00211u_color_2.png" alt="image"><img src="/images/image_pyramids/00211u_color_2_preview.png" alt="image"></a>
    <a href="/images/image_pyramids/00093u_color_2.png" alt="image"><img src="/images/image_pyramids/00093u_color_2_preview.png" alt="image"></a>
    <figcaption>Multi-scale aligned glass plate negatives.</figcaption>
</figure>

## Experimental Analysis

My alignment algorithms resulted in crisp photos overall, as seen above. There was one picture in my test set that didn't do well with my edge evaluation. This could be because there weren't strong edges to match in the photo. It did much better with a simple brightness alignment.

<figure class="half">
    <img src="/images/image_pyramids/00800v_color.jpg" alt="image">
    <img src="/images/image_pyramids/00800v_color_2.jpg" alt="image">
    <figcaption>(Left) Aligned with edges. (Right) Aligned with brightness.</figcaption>
</figure>

In the picture of the man in the garden, there is quite a bit of noticeable haziness. This could be due to the camera being unfocused or overexposure from the bright outdoor lighting. The stained glass example also has some artifacting upon close inspection. The yellow channel seems shifted, which could be caused by indistinct boundaries in the yellow channel.

## Extra Credit

I attempted cropping and edge alignment for extra credit.

Edge alignment involved applying a Sobel filter from the SciPy package on the image before the differences between two channels were calculated. Details of how it was incorporated into the algorithm in described in the single-scale alignment section above. Without edge alignment, there was a considerable amount of blur in each image. You can see a sample below.

<figure>
    <a href="/images/image_pyramids/without_edge.jpg" alt="image"><img src="/images/image_pyramids/without_edge_preview.jpg" alt="image"></a>
    <figcaption>So blurry that it looks like an earthquake.</figcaption>
</figure>

Next, I implemented auto-image cropping. This was one of the most challenging parts of the lab because the dark border around each image wasn't uniform. Some of the borders were even jagged or slanted. I cut off the dark border in two steps: crop the left and right borders from the entire original image and then crop the tops and bottoms of each separated channel. I needed two separate steps to ensure a uniform cut across all three channels since the sizes of each image needed to match in order to use my evaluation function.

To crop the left-right borders, I need to determine if an area of the photo is black, increase the contrast, round each pixel to either black or white, sum up the number of white pixels are in each column. If the number of white pixels doesn't exceed a threshold (30% white), then that column is a border. As soon as I find a column that's whiter than the border, I set that as the cut off point and crop the image. I allow for white specks within 2% of the width and I don't allow the border to exceed 10% of the image width overall. The same process was used for the top and bottom borders, but with the rows and cols switched. The images look considerable neater without the dark borders. Examples of uncropped borders can be seen on the Library of Congress's website.

I feel like I can keep tinkering with this lab forever. There are so many small changes and optimizations to be made. This project was a great learning experience. A takeaway I have was learning proper [Python styling](https://google.github.io/styleguide/pyguide.html) and project organization. I hope to learn even more Python throughout the semester.

### Thanks!

<figure>
    <a href="/images/image_pyramids/dog.gif" alt="image"><img src="/images/image_pyramids/dog.gif" alt="image"></a>
    <figcaption>Time lapse of a dog's nap -- what could it be dreaming of?</figcaption>
</figure>