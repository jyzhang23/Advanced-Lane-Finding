## Advanced Lane Lines Project Writeup

### Submitted by Jack Zhang
#### 12/10/2017

---

**Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./output_images/cal1_undistort_comparison.png "Calibration1 Undistorted"
[image2]: ./output_images/test1_undistort_comparison.png "Test1 Undistorted"
[image3]: ./output_images/test4_threshold.png "Threshold Comparison"
[image4]: ./output_images/gradx_low_thresh_7px.png "Gradx threshold test"
[image5]: ./output_images/schannel_low_thresh.png "S threshold test"
[image6]: ./output_images/mask_threshold.png "Mask threshold"
[image7]: ./output_images/warp_straight1.png "Warped image"
[image8]: ./output_images/line_fitting.png "Line detection and fitting"
[image9]: ./output_images/test4_final_output.png "Final output"
[video1]: ./project_video_output.mp4 "Video output"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the IPython notebook located in "Advanced_lane_lines_project.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection. The chessboard pattern I am looking for is a 9x6 pattern.

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the calibration image *calibration1.jpg* using the `cv2.undistort()` function and obtained this result: 

![Chessboard undistorted][image1]


### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![Test1 undistorted][image2]

This is *test1.jpg*. It looks very similar to the original, but you can see the difference on the hood of the car, and the white car on the right.

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds, to generate a binary image (code cell 3 in the notebook).  Here's an example of my output for this step, on *test4.jpg*. 

![Threshold comparison][image3]

The gradient threshold used a Sobel filter in the x direction, with threshold values of (30,200) and a kernel size of 7. These values were determined experimentally by comparing different thresholded images like this:

![Gradx threshold test][image4]

The color threshold was done by converting the image to HLS color space, and extracting the S channel with a lower threshold value of 170. Different threshold levels were also explored experimentally.

![S threshold test][image5]

Finally, I applied an ROI mask as shown in the following figure. This mask is to remove some noise in the binary image, and is larger than the region transformed into a top-down perspective view.

![Mask threshold][image6]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform is located in the 7th code cell of the Jupyter notebook. The source (`src`) and destination (`dst`) points are hardcoded according to:

```python
src = np.float32([[236,690],[562,470],[718,470],[1070,690]])
dst = np.float32([[300,720],[300,50],[980,50],[980,720]])
```
These values were determined manually by selecting points on a test image with straight lines (*straight_lines1.jpg*), and selecting points that bound a rectangle in the original image.

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 236, 690      | 300, 720      | 
| 562, 470      | 300, 50       |
| 718, 470      | 980, 50       |
| 1070, 690     | 980, 720      |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel and straight in the warped image.

![Warped image][image7]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Once I had a top-down thresholded image of the lanes, I identified the lane positions by using a convolution approach (code cell 8 in the Jupyter notebook). First, the image was divided into 6 vertical sections of 120 pixels. A convolution window (50 pixels wide, all 1) was convolved across each vertical section of the image, on both the left and right half of the image, to extract the location of the lane lines. If a convolution result was zero (there were no bright pixels in the warped image), the previous window location (the one below it) was used. This was important to add, otherwise the leftmost convolution result would return as the default. From these 6 positions, a 2nd order polynomial was fit for both the left and right lane lines. The following figure shows the output of this algorithm on the test image, *test4.jpg*. 

![Line detection and fitting][image8]

At the end of the Jupyter notebook (code cells 9-12) is the code for processing on the videos. For the videos, I implemented some additional features. I defined a `Line` class to store information for both the left and right lines, and implemented a running average of 10 frames to average the lane locations. After the first 10 frames, the `Line` objects contains the window locations of the previous 10 frames. This allows a targeted search for the next lane locations, around the average location of the previous 10. Also the polynomial is now fit to the average of the last 10 frames lane detections, which smoothes out the lane detection significantly. 


#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

In code cells 11-12 of the notebook, I calculate the radius of curvature of the lanes and the position of the vehicle with respect to the center. The radius of curvature is calculated in code cell 12 by using the 10 frame average lane location, converting the units from pixels to meters, and fitting a 2nd order polynomial to those points. The radius of curvature is then calucated using the coefficients of that fit. This is done for the left and right lines, and the two are averaged to get a final radius of curvature. 

The center position is calculated (code cell 12) by first calculating the center of the lane lines, given by the 10 frame average of the lane line locations, and subtracting that number from 640, which is the center of the image. This number is converted from pixels to meters, and expressed as centimeters in the final image output. In this convention, a position number represents deviation to the right of the lane center, and negative number is deviation to the left of the lane center.

These values are passed to the function `draw_lanes` (code cell 11), where they are drawn on top of the final image.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

In code cell 14 of the notebook, I read in a test image and run the whole image process pipeline. Here is the result on *test4.jpg*:

![Final image output][image9]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video_output.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The biggest issues I faced in this project was features that appeared in the thresholded image from contrast on the road not due to the lane markers. This occured from imperfections and color changes in the road, as well as shadows. While the S channel performed reasonably robustly in these light conditions, sometimes there were still lots of extra features that obscured the lane lines. To deal with this, I implemented the 10 frame running average, which helped prevent the lane lines from jumping so rapidly.

To make the pipeline more robust, I would explore additional color spaces and thresholding techniques. Perhaps I would add 2 additional channels that specifically look for white and yellow lines to augment the binary image. As far as I know, lane lines are always either yellow or white. 

Another possibility is to add a weighting scheme to the two lane lines. For instance, if one line is detected with more confidence than the other, then it could be weighted more confidently in terms of the polynomial fit, and the other one could be extracted since the distance between 2 lane lines are pretty constant. 
