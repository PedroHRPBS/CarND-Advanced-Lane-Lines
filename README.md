## Writeup

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

[image1]: ./output/calibrate.png "Calibrate"
[image2]: ./output/undistort.png "Road Transformed"
[image3]: ./output/perspective.png "Warp example"
[image4]: ./output/colorpipeline.png "Binary"
[image5]: ./output/slidingwindow.png "Fit Visual"
[image6]: ./output/result.png "Output"
[video1]: ./output/project_video_output.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the 3rd. 4th and 5th code cells of the IPython notebook located in "./Advanced-Lane-Finding-project.ipynb". 

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I loaded the `dist` and `mtx` from a pickle file that I created after calibrating the camera. Using these values and `cv2.undistort` I implemented the first step of my pipeline, as you can see on the following image:
![alt text][image2]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

First, I inverted the order of the operations on my project. Because I found it to be easier to identify lanes using gradients after the perspective transformation.

The code for my perspective transform includes a function called `perspective()`, which appears in 8th code cell of the IPython notebook.  The `perspective()` function takes as inputs an image (`image`).  Inside this function, I chose the hardcode the source and destination points in the following manner:

```python
   src = np.float32([[200, img_size[1]-50],
                      [img_size[0]//2-55, img_size[1]//2+90],
                      [img_size[0]//2+55, img_size[1]//2+90],
                      [img_size[0]-200, img_size[1]-50]])

    dst = np.float32([[offset, img_size[1]-1],
                      [offset, 0],
                      [img_size[0]-offset, 0],
                      [img_size[0]-offset, img_size[1]-1]])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 200, 670      | 400, 719      | 
| 585, 450      | 400, 0        |
| 695, 450      | 880, 0        |
| 880, 670      | 880, 719      |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image3]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color, gradient thresholds and area of interest to generate a binary image.  
For the color transform, I chose to use HLS color scheme and focused on the S channel, because it is more robust on light variations.  
For Gradient Threshold I focused on the Sobel gradient in X direction, because as the lanes are more vertical, it would highlight them better.  
Besides that, I also selected an area of interest so I could avoid having misinformation on the sides.
The code for that can be seen at the 10th code cell of the Jupyter Notebook. Here's an example of my output for this step.  

![alt text][image4]


#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

To achieve that, on code cell #12, I utilized a histogram + sliding window algorithm.
The function of the histogram, is to sum all the pixels from the bottom half of the image, in the vertical direction. Doing so, we can detected where two locations with more pixels, and that represents our two lines.
The sliding window algorithm is useful to detected the whole line, so we can fit a polynomial to it and have its equation. With that its easier to calculate the curvature of the curve.  
The final result can be seen on the next picture:

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this on the 15th code cell of the Jupyter Notebook. To do so, first I got my polynomial equation that represents my lines and we that, I applied the curvature formula from the classroom:

```
left_curverad = ((1 + (2*left_fit_cr[0]*y_eval + left_fit_cr[1])**2)**1.5) / np.absolute(2*left_fit_cr[0])
right_curverad = ((1 + (2*right_fit_cr[0]*y_eval + right_fit_cr[1])**2)**1.5) / np.absolute(2*right_fit_cr[0]) 
```
It is good to highlight that in this function, we are also performing the conversion from pixel to meters, so we can have a correct and readable value of curvature.

```
ym_per_pix = 30/720 # meters per pixel in y dimension
xm_per_pix = 3.7/700 # meters per pixel in x dimension
```
To perform the calculation of vehicle position, we had to consider that the camera was fixed of the middle of the vehicle. With that, we calculated the values of the poplynomials on the bottom of the image and with that, we found the middle of the lane.

```
bottom_y = 669
vehicle_center = 640
    
l_fit_x_int = left_fit_cr[0]*bottom_y**2 + left_fit_cr[1]*bottom_y + left_fit_cr[2]
r_fit_x_int = right_fit_cr[0]*bottom_y**2 + right_fit_cr[1]*bottom_y + right_fit_cr[2]
    
lane_center_position = (r_fit_x_int + l_fit_x_int)/2
vehicle_pos = (vehicle_center - lane_center_position) * xm_per_pix
```

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step on the code cell #16,`plot_lane_lines()`.  Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./output/project_video_output.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  
