## Advanced Lane Finding Project

---

**Henry Yau**
**Oct 16,2017**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

<<<<<<< HEAD
This readme/writeup refers to the iPython (Jupyter) Notebook file `AdvancedLaneProj4.ipynb`, however the code is also implemented in a Python file `laneOverlayCreate.py`
=======
This readme/writeup refers to the iPython (Jupyter) Notebook file `./src/AdvancedLaneProj4.ipynb`, however the code is also implemented in a Python file `./src/laneOverlayCreate.py`. Please view AdvancedLaneWriteup.ipynb to see the equations correctly displayed.
>>>>>>> 1134064415e540be86256a041ff161030880cb6b

[//]: # (Image References)


[image1]: ./write_up_images/undistort_calibration.png "Undistorted"
[image2]: ./write_up_images/preprocess_pipeline.png "Image pipeline"
[image3]: ./write_up_images/undistorted_difference.png "Undistorted difference"
[image4]: ./write_up_images/unwarped_warped_parallel.png "Perspective transform"
[image5]: ./write_up_images/historgram.png "Histogram"
[image6]: ./write_up_images/lanesWindow.png "Lane margins"
[image7]: ./write_up_images/laneMask.png "Perspective transform"
[image8]: ./write_up_images/laneComplete.png "Perspective transform"





---

### Camera Calibration

Images captured using real lensed cameras will have distortions caused by the curvatures of the lenses (radial) and imperfect centering of the lenses (tangential). The distortions are most pronounced near the periphery of the image where lines which are straight in the real world appear as curved in the image. These distortions are removed so that we can use a so called pinhole camera model and safely make assuptions about spatial relationships of features found in the image.

The distortion model used in OpenCV was first described by Brown in a 1966 paper entitled, "Decentering Distortion of Lenses". This model is also called the Brown-Conrady model. Five distortion coefficients are used in the model, three for radial $k_i$ and two for tangential $p_i$. The corrected $x$ and $y$ coordinates are shown below where $r$ is the distance of $(x,y)$ to the center of $(x_c,y_c)$:

$$x_{corrected} = x(1+k_1r^2+k_2r^4+k_3r^6)+(2p_1xy+p_2(r^2+2x^2))$$
$$y_{corrected} = y(1+k_1r^2+k_2r^4+k_3r^6)+(p_1(r^2+2y^2)+2p_2xy)$$

Higher order radial distortion terms up to $k_6$ can be used in OpenCV. In addition to the distortion effects, we can compute the camera matrix which maps the real world to the distorted coordinates. This allows us to map image space distances (eg. pixels) to real world distances (eg. mm). 

Thankfully OpenCV provides a built in implementation for calibration. The following steps are implemented in the first cell of the iPython Notebook (`AdvancedLaneProj4.ipynb`). The function `cv2.calibrateCamera()` takes a set of 3D points in the world (`objpoints`) and a corresponding set of 2D points in image space(`imgpoints`) and returns the camera matrix and distortion coefficients. A sufficient number of calibration images in the form of checkerboard patterns with 9x6 internal corners are used to create a well posed problem. The function `cv2.findChessboardCorners()` returns the image space location of each of the internal corners and is used to produce the list `imgpoints`. The list of 3D points `objpoints` is artificially created to be squares of 1x1 units in the x-y plane with the z-coordinate set to zero. Since the real world units are not used, the projection matrix does not provide us with real world distance measurements.

Using `cv2.undistort()` with the camera matrix and distortion coefficients we can undistort any image taken with the same camera. A test image is shown below before and after distortion correction. The "barrel" effect is noticable on the left image whereas on the right, the checkerboard appears to be straight.

![alt text][image1]

### Image Pipeline

#### 1. Image preprocessing
A set of small image processing functions are implemented in the second cell of  `AdvancedLaneProj4.ipynb` under "Image Processing Routines". The first routine is `undistort_img()` which calls `cv2.undistort()` using a precomputed camera calibration. 

The next few functions, `sobel_thresh()`, `mag_thresh()`, and `dir_thresh()`, are variations of edge detection using the Sobel operator. The function `color_thresh()` transforms an image to HLS colorspace and attempts to highlight yellow and white. These four routines are used in combination in the function `processImage()` to create a binary image which enhances lane markings. 

The threshold parameters were then tweeked until the binary output image looked satisfactory. Using this routine on a test image produces the results below. 

![alt text][image2]

The undistorted image may appear to be the same as the original image, however by overlaying the images and computing the difference, we can spot the effects of distortion. For example the lane marker on the right is in an exitrely different position.

![alt text][image3]


#### 2. Perspecive transform

We wish to now transform the image from the undistorted imagespace to a bird's eye view of the road. This may be done in several ways but here in the fourth cell of the iPython Notebook we manually determine the transformation using an image of a relatively straight road. We know the lane lines are parallel in the real world, so we would like the transformation to produce an output where the lane lines are also parallel in image space.

A trapzoid enclosing the lane lines in the undistorted image corresponds to a rectangle enclosing the lines in the warped bird's eye view image. A quality mapping is manualLy determined to be:


| Source        | Destination   | 
|:-------------:|:-------------:| 
| 190,706      | 190,706       | 
| 608,440     |190,0      |
| 670,440     | 900,0     |
|1150,706      | 900,706       |


OpenCV provides functions to compute the transformation matrix using the source and destination points with `cv2.getPerspectiveTransform()`. Using the transformation matrix with the function `cv2.warpPerspective()`, we produce the output below. Overlayed on the image are the trapazoid and the corresponding rectangle. 

![alt text][image4]


#### 3. Line line finding

Initial lane line finding is illustrated in the iPython cell labeled "Finding lane lines, initially use sliding window" and also in the complete image pipeline function `process_lane_overlay()` which performs the operation on a `Line()` class. A histogram of the bottom 1/5 of the image is first computed to find a good initial starting place for the sliding window. The two maxima correspond to the two lanes, so we use the corresponding x-positions as the start of the lanes. This histogram is shown below.
![alt text][image5]

A window is centered as this position and the number of "on" pixels within the window is counted. If it is above a certain threshold, the pixels are put into a list and the average x position of the "on" pixels is used for the center of the next window. The window is then moved up and the process is repeated until the window reaches the top of the image. We now have a list of "on" pixels. An interpolating second order polynomial can be found using `np.polyfit()` on the list of "on" pixels for each lane.

The computation of the sliding window is slow but fortunately does not need to be performed often. So long as the lane line does not deviate significantly from the previous frame, we only need to search within a neighborhood close line determined by the sliding window. This neighborhood is shown below in green and is illustrated in the cell "Use previously found lanes to limit regions for searching current lanes" and again within the actual pipeline in `process_lane_overlay()`.

![alt text][image6]

If however there is some discrency between the two lanes, such as wildly differing radii of curvature or a computed distance from center greater than one meter, then it recomputing using the sliding window should be done. This is done by calling `reset()` for the `Line()` class.

The image above illustrates a problem with using a single image to determine the lane lines. The right lane is dashed and so at this particular frame is lacking information near the bottom. The resulting polynomial has a greater curvature than the actual road. Therefore it is beneficial to keep track of the history of the lane line values and perform an average. This is done in the `Line()` class in the function `add_new_data()` which uses a queue of the previous evaluated x positions of the line. At this time we can also perform a low pass filter by excluding extreme lane estimations. This is done by comparing the average radius of curvature to radius of curvature of the current polynomial.


#### 4. Radius of curvature

The radius of curvature $R_{curvature}$ is the inverse of the curvature $K=\frac{d\theta}{ds}=\frac{d^2y}{dx^2}$. For a polynomial $x=f(y)=P_0y^2 + P_1y+P_2$ the radius of curvature can be written as :

$$R_{curvature} = \frac{(1+(2 P_0(y_{ppm}^2/x_{ppm})+P_1(y_{ppm}/x_{ppm}))^2)^{(3/2)}}{2P_0(y_{ppm}^2/x_{ppm})}$$

where $x_{ppm}$ and $y_{ppm}$ are conversion factors screen space to world coordinates in meters. Using  https://mutcd.fhwa.dot.gov/htm/2009/part3/part3b.htm we find the gap between lane markings to be 30 feet (9.144m), in the warped image this corresponds to 95 pixels high in the y-axis. The lane width is 3.7m which corresponds to 419 pixels wide in the x-axis. This leads to a scaling factor of $y_{ppm} = 95/9.144$ and $x_{ppm} = 419/3.7$.

The routine is implemented in `findRoadCurvature()`.

#### 5. Final image

The function `draw_overlay()` composites an image of the estimated lane using the left and right lane marker polynomials onto the undistorted image of the road.

The warped estimated lane looks like the image below:
![alt text][image7]

Below is the unwarped estimatd lane using the inverse perspective matrix and overlayed on the undistorted image:
![alt text][image8]


---

### Pipeline (video)


Here's a [link to video result](./output_video/project_video_output.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Not knowing the actual curvature of the road makes it difficult to determine how far ahead to look. For example, if the road is perfectly straight, one could ahead perhaps 40m. However if the road is curving significantly, using that 40m window could mean the window contains road side objects which could be misinterpreted as lane lines.

One solution is to simply not look as far ahead but using knowledge of the camera transformation it may be possible to create a solution which looks far ahead even on curvy roads. Using dead recogning we can construct a polygon around the anticipated road region then transform to the warped bird's eye view. One could also crop regions of the warped image which we are confident do not lie in the road.

One glaring issue is the speed and memory usage of the method. The lane line polynomials are evaluated for every y-axis pixel (720) and 15 are stored in the history for each line. The history is also handled in a very inefficient manner requiring pushing and popping for every frame. A more efficient static array with a moving index could easily be implemented. As it is, the proceedure can only handle less than three frames per second. To achieve real time lane detection, the image size should be scaled down and perhaps use only grayscale images. It is also not necessary to process every frame.

The two challenge videos each present different issues to overcome. In one video the road has patchwork which has high contrast between asphalt and concrete that confuses the lane line detection algorithm. The other video has a winding road with tree shadows, both of which make determining the lane lines difficult. To robustify the processes one could perform additional preprocessing steps such as color correction and contrast adjustment.
