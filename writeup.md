# **Finding Lane Lines on the Road**

## Writeup

---

**Finding Lane Lines on the Road**

The goals / steps of this project are the following:
* Make a pipeline that finds lane lines on the road
* Reflect on your work in a written report


[//]: # (Image References)

[gray]: ./examples/grayscale.png "Grayscale"
[blurred]: ./examples/blurred_gray.png "Blurred Grayscale"
[canny]: ./examples/canny.png "Canny Edge Detection"
[mask]: ./examples/region_mask.png "Region Mask"
[masked_edges]: ./examples/masked_edges.png "Masked Edges"
[hough]: ./examples/hough_lines.png "Hough Transform"
[lines]: ./examples/lane_lines.png "Lane Lines"
[out]: ./examples/out.png "Final Output"
[shadowed]: ./examples/shadowed.png "Shadowed Lane"
[no_yellow]: ./examples/no_yellow_lane.png "No Yellow Lane"
[challenge]: ./examples/challenge_out.png "Challenge Output"

---

### Reflection

### 1. Describe your pipeline. As part of the description, explain how you modified the draw_lines() function.

The basic pipeline consists of 5 steps, with the goal of stripping out all possible noise and
isolating the left and right lane markers in the image/video. The first 3 steps involve
converting the image to grayscale to strip out any color information, applying Gaussian blur
to remove noise and applying Canny edge detection.

![alt text][gray] ![alt text][blurred] ![alt text][canny]

This gives us all the potential lane edges in the image (and a bunch of other stuff we don't need). We know that the camera is mounted in a fixed position in the car, which means that the horizon will be somewhere in the middle of the image and the road will be on the bottom
half of it, converging towards the center. We can isolate that by creating a trapezoidal mask
and removing all the edges that fall out of the mask region. Applying Hough transform afterwards
with the right parameters gives us the lane lines in the image.

![alt text][mask] ![alt text][masked_edges] ![alt text][hough]

It's all pretty straight-forward so far. Now we need to extrapolate all the line segments
that we just received to map out the lane boundaries. We can do that by calculating the slope for each line (`slope = (y2-y1)/(x2-x1)`) and sorting them out based on whether they are positive or
negative. Given that the origin is on the upper left-hand side of the image, the right lane will have a positive angle/slope and the left lane will have a negative slope.  By averaging these slopes and the points on each side, we can use the results to fit a line through each lane.
It took me forever to figure this out (flunked Algebra class, don't ask), but we can then draw the lanes using the point-slope form (`(y - y') = m(x - x')`), given what we have plus our region
mask's y-coordinates as our y values. This gives us:

![alt text][lines]

Super-imposing these lines on top of the original image allows us to check if the lines are
a reasonable fit. I've also added green lines to let us compare the results from the Hough
transform step.

![alt text][out]

The challenge video provided some interesting challenges (heh). First, the lanes are now curved
and there are sections of the road where shadows trip up edge detection. The hood of the car also
adds some additional noise to the image. There wasn't much I can do with the curves yet,
which was all right for now since it's only causing issues towards the horizon. The shadows
causes the yellow lane to be indistinguishable with the main road, so we need to bring some color
information back. You can also notice some horizontal lines managing to sneak themselves inside the region, which causes our final lane markers to skew towards horizontal.

![alt text][shadowed] ![alt text][no_yellow]

It sounds like a hack, but by adding new edges based on color information, we can preserve some
of the lines that were lost from the grayscale conversion. We filter out the yellow lane by
converting the image to its HSV form and using a limited hue and saturation range as thresholds.
Adding the filtered image values to the Canny edges gives the Hough transform more lines to latch
on to. With regards to the lines created by the hood, we take out all detected lines outside a
certain angle range (in this case, roughly between 21 and 56 degrees). With these changes,
we create a reasonable output:

![alt text][challenge]

### 2. Identify potential shortcomings with your current pipeline

There are probably a lot of potential short-comings, but first thing that comes to mind is how sensitive edge detection is to shadows and noise. The pipeline assumes a best-case scenario where we're on a wide highway, there are no cars upfront, no objects are obscuring the lane markers, and there is a reasonable amount of lighting/daylight that allows edge and line detection to do its work.

Right now it also doesn't handle any curved roads well at all. The highway provides a large enough turn that we can treat the most of the lane as a straight line, but for tight hairpin turns for example, this won't work.

As far as range is concerned, it's only capable of picking up lanes that are approximately 3 cars in length, which doesn't sound sufficient, especially in terms of being able the anticipate sudden changes in road condition.

We're also filtering out certain lines (like horizontal ones) as noise, but there are certain cases where this is important, like lines that mark the end of a road for example. In these cases we might want these lines as additional input to the car's decision-making process.


### 3. Suggest possible improvements to your pipeline

Well, limiting the scope of improvements to just the lane detection quality, we can address the issue of not being able to adapt to turns by doing some sort of curve fitting (which is way beyond me right now). Assuming lane line features don't change abruptly, we can also use previous lane information to predict the lanes in the subsequent images, avoiding the "oh no there's a car on top of the line, there are no more lines! noooo! _crash_**" situation.

If we can detect vehicle silhouettes (or any kind of non-lane silhouettes for that matter), we can also subtract those from the image to reduce noise.
