# **Finding Lane Lines on the Road** 

The goals / steps of this project are the following:

* Create a pipeline that finds lane lines on the road
* The pipeline should find exactly two lanes (left and right) which are marked as continuous lines throughout the region of interest


[//]: # (Image References)

[image1]: ./images/region_mask.png "region mask"
[image2]: ./images/pipeline.png "pipeline"
[image3]: ./images/hough.png "hough"
[image4]: ./images/tree_noise.png "tree noise"
[image5]: ./images/color_converted_before.png "grayscale"
[image6]: ./images/color.gif "color animation"
[image7]: ./images/color_converted_after.png "color converted"
 
---


### 1. Description of the Pipeline

As described in the CarND-LaneLines course, we start with the following simple steps to find appropriate edges that can be associated with lane markings:

* we first convert the image to grayscale to facilitate the detection of white lane markings
* we blur the image using a Gaussian filter to average out picture defects such as noise
* we use Canny edge detection to find regions of high color gradient
* as thresholds for the Canny algorithm I chose to use [Otsu's method](https://en.wikipedia.org/wiki/Otsu%27s_method) instead of finding fitting parameters by trial-and-error, i.e. edge pixels with gradients above Otsu's threshold are always marked while edge pixels with gradients lower than `0.5*Otsu` are always suppressed
* finally a region mask is applied to the image to restrict the edge detection to the region right in front of the car:

![Region Mask][image1]

The edge detection pipeline looks like this:

![Pipeline][image2]

Now that the edge pixels are identified, the next step is to use the [Hough transform](https://en.wikipedia.org/wiki/Hough_transform) on our edges to find potential lines of a certain length.

To fine-tune the line detection we use the following parameters:

```python
## parameters for the filtering algorithms
# Size of the Gausian blur kernel
blur_kernel_size = 5
# Canny thresholds (from Otsu thresholding)
canny_low_threshold = 65
canny_high_threshold = 130
# distance resolution for the line accumulator
hough_rho = 2
# angular resolution
hough_theta = np.pi/180
# vote threshold
hough_threshold = 40
# minimum length of a line
hough_min_line_len = 40
# aximal gap size in a detected line
hough_max_line_gap = 20
```

The Hough transform now yields the following result:

![Hough][image3]

As we can see, while the lane markings are clearly detected we still don't have continuous markings for the left and right lane. We are also still picking up lines that should not be associated to any lane markings.

To filter out additional lines and detect the line segment that belong to the left and right lane respectively I chose the following approach:

1. Parametrize all detected lines by angle $\theta$ and intersection $b$ with the x-axis, also compute the length of the detected line
2. Neglect all lines that do not intersect the bottom of the frame. They should not belong to any ego lane markings.
3. Lines that intersect on the left-hand side of the frame are associated with the left lane marking and vice versa.
4. Compute a weighted average of the parameters $\theta$ and $b$ with weights giving by the length of the line coming from Hough's algorithm
5. Draw the line up to the maximum height coming from the highest y value in all considered lines

The finished pipeline now produces continous lane markings:

<details><summary>Finished Pipeline</summary>
<p>
<video width="640" height="360" controls src="test_videos_output/solidYellowLeft.mp4"  type="video/mp4"/>
</p>
</details>

### 2. Challenge Trace

Additionally, another video from a different car and camera was provided to probe the robustness of our pipeline:

<details><summary>Challenge Video</summary>
<p>
<video width="640" height="360" controls src="test_videos_output/challenge_1.mp4"  type="video/mp4"/>
</p>
</details>
<br/>

As we can see, there are several issue with this trace:

1. the curvature of the street
2. the properties of the video output have changed, i.e. we ought to recalibrate the pipeline
3. yellow lane markings are not tracked as well as white ones (because of the grayscale conversion)
4. the contrast on the bright segment of the street seems very low
5. the street's surface changes several times, which introduces new lines in the image
6. erratic noise is created from the tree's shadow
![Tree Noise][image4]

For this project we will neglect the first point and still try to create a linear fit. 
To address points 5 and 6 I propose to filter the lines more strictly, i.e. if the angle of the line is to steep it should be discarded. To reduce the influence of noisy backgrounds we should take into account the
history of our detected line and only accept new lines which don't differ to much from the moving average of the lane over the last frames.

<details><summary>Challenge Video with Lane History</summary>
<p>
<video width="640" height="360" controls src="test_videos_output/challenge_2.mp4"  type="video/mp4"/>
</p>
</details>
<br/>

Here a yellow line means that the lane was not changed from the last frame because the new lane differed too much from the average. After 10 frames the line turns and red and is reset.
As we can see, the line is already tracked much more smoothly. However, we still lose track of the lane on the bright surface because the contrast between the street and the yellow lane is too low after rgb to grayscale conversion.

![Bright surface][image5]

One possible remedy is to use an additional color filter for white and yellow lanes. We use the following segments in [HSV Color space](https://en.wikipedia.org/wiki/HSL_and_HSV):

```python
light_yellow = (20, 100, 200)
dark_yellow = (33,255,255)
light_white = (0, 0, 180)
dark_white = (255,30,255)
```
![Filtered colors][image6]

With a filter restricted to only these colors, the yellow line becomes clearly visible:

![Color filtered image][image7]

The two lanes are now tracked almost continuously throughout the trace:

<details><summary>Challenge Video with Color Filter</summary>
<p>
<video width="640" height="360" controls src="test_videos_output/challenge_3.mp4"  type="video/mp4"/>
</p>
</details>

### 3. Potential Shortcomings and Possible Improvements


There are still several issues with our current pipeline. 

For one, we still don't take into account the curvature of the lane (and ground). Instead of straight lines we should rather look for polynomials or clothoids for a better fit with the video data. This could possibly be accomplished by performing a perspective transform on the road segment.

Secondly, the fine tuning of the Canny and Hough parameters heavily depend on the given video. They should be dynamically adjusted to the given environment, e.g. night time, snow, fog, which all require a different tuning of the parameters.

Thirdly, the filtering of the relevant lines could be elaborated. So far, all detected lines that fit some crude restrictions are accounted in the moving average. However, they might not all come from the same lane marking.

Lastly, if the lane markings on the road are too weak to confidently detect them we can use other means to identify lanes, for example other car trajectories or even the road boundaries.