## Writeup Template

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

[image1]: ./output_images/undistored11.jpg "Undistorted"
[image2]: ./output_images/undistorted_road1.jpg "Road Transformed"
[image3]: ./output_images/threshold1.jpg "Binary Example"
[image4]: ./output_images/straight_lines_perspective.png "Straight line Image for Perspective Transform"
[image5]: ./output_images/perspective_after_1.jpg "Warp of straight line example"
[image6]: ./output_images/perspective_after_2.jpg "Warp of curved line example"
[image7]: ./output_images/centroid1.jpg "Fit Visual"
[image8]: ./output_images/road/road1.jpg "Output"
[video1]: ./annotated_project_video_1.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation. All code references will point to mark-down headings in the provided Jupyter notebook.  

---

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The camera calibration is done against images of a 9x6 chessboard.
The first step in camera calibration is to collect the object points of the inner corners 9x6 chessboard. The object points are (x,y,z) cartesian co-ordinates with z being 0 since the images are two-dimensional.
The provided example images of the chessboard are used to collect the cartesian co-ordinates of the inner corners in the images. This is done through the openCV library methods `findChessboardCorners` and `drawChessboardCorners`.
The camera matrix and distortion co-efficients that map the object points to the image points are then calculated using the openCV method  `calibrateCamera`.

The distortion co-efficients are used to undistort the images. Here is an example:

![Undistored image][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

As mentioned before, the distortion co-efficient is used to undistort the camera images of the road. Here is an example:
![Undistored Road Image][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

Code Section: ```Perspective Transform```  

The thresholding is composed of the following elements:
- V channel thresholding in an HSV converted image
- S channel thresholding in an HSL converted channel
- Gradient magnitude
- Gradient direction
- X-Gradient
- Y-Gradient

The above elements are combined as follows:  
`((X-Gradient && Y-Gradient) || (Gradient-Magnitude && Gradient-Direction)) || (V-Threshold || S-threshold)`

An example of the resulting image is:
![Binary Image][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

Code-Section: ```Perspective Transform```  
The perspective trasnsform is performed using the openCV function `getPerspectiveTransform`.
An image with straight lanes is inspected and four cartesian points forming a trapezoid are selected. These are then mapped on to four desired points forming a rectangle. 


Source and Destination points used:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 593, 450      | 300, 0        | 
| 684, 450      | 950, 0        |
| 1095, 720     | 950, 720      |
| 195, 720      | 300, 720      |

![Perspective Transform Input][image4]

The resulting perspective transform is verified by checking that
- The straight lines are straight and parallel in the transformed image
![Transformed Straight line image][image5]
- The curved lines are parallel in the transformed image
![Transformed curved line image][image6]


#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Code Reference: `Lane Detection` Class: `Tracker`

The window-centroid approach described in the course is utilized to detect lane pixels.
The method consists of dividing the image into 9 horizontal slices each of width 25 pixels.
The y co-ordinates are dictated by the image height and the slice height (80 px)
The x co-ordinates are computed using the following approach:
- Identify a `target region` in the image slice
- Reduce the image to one-dimension (x co-ordinate)  by summing the pixels in the y-direction of the slice
- Apply one-dimensional discrete convolution on the `target region`
- The index corresponding to the maximum value (meeting a threshold value) in the convolution is the x-co-ordinate (adjusted for center since the index of max value will correspond to the right edge of the convolution signal window). If the maximum value does not meet the threshold value, previously found centroid value is used.

The target region for the first horiziontal slice is:  
y: Bottom quarter of the image. The bottom quarter is chosen based on the assumption that the rest of the image will contain features of lesser interest like landscape.  
x: Left half of the image for left lane and right half of the image for the right lane.  

The target region for subsequent horizontal slices are:  
y: Window height  
x: Previous centroid's `x` +/- 100 pixels  

This results in 9 window centroids in x-y cartesian co-ordinates representing points on the left and right lanes respectively.
For subsequent images, the values are smoothed using a moving average of 11 centroids.

An example image with window centroids:
![Window Centroids][image7]

These points are fitted to a second-degree polynomial to obtain the lane line equation. (Code Reference: `Tracker::fit_window_centroids`)
The resulting lane line is plotted onto the original image using the inverse perspective transform.

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

Code Reference: `Lane Detection` Class: `Tracker`

The radius of curvature is computed using the arithmatic formula utilizing the first-order and second-order derivates of the lanle line equation. The lane line equation should however represent lines in real world space and not the pixel space
Using the given ratios of pixels to lane lines, a new set of polynomials are fitted in the real world space. (Code Reference: `Tracker::fit_window_centroids`).

Ratio of meters to pixels:  
`Y: =30/720  X: 3.7/700`  

The vehicle position is computed as an offset from the lane center assuming the camera was mounted on the center of the vehicle. As such, it is the difference between the x co-ordinates of the center of the computed left and right lane at the bottom of the image and the half of the image width. The resulting pixel value is scaled to real world terms by the provided ratios. 

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.
This operation is done in `Tracker::polyfill_road`.
Here is an example:

![Plotted Lane Lines][image8]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result][video1]

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?
- Identifying the points to perform the perspective transform was done manually.
- The gradient and color threshold tuning was a manual process
- Lane markings within the lane lines would throw off the lane positioning algorithm
- Mix of road surfaces causing color gradients would throw off the lane positioning algorithm
- The algorithm could be thrown off by the image quality of the camera. For example, debris on the camera could compromise the image quality.
- The algorithm does not use the lanes identified in the previous vide frame These could be re-used to perform a more targeted search for lanes.
