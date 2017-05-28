# **Advanced Lane Finding Project Write-up** 

### This is a write up on the Advanced Lane Finding project

---

### **Goals of the Advanced Lane Finding Project**

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./examples/undistort_output.png "Distorted and Undistorted images"
[image2]: ./examples/BinaryThreshold.png "Binary Threshold Image"
[image3]: ./examples/warped_straight_lines.png "Warp Example"
[image4]: ./examples/color_fit_lines.jpg "Fit Visual"
[image6]: ./examples/example_output.jpg "Output"
[video1]: ./project_video.mp4 "Video"

---

### *Camera Calibration*

**1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.**

The code for this step is contained in the **_caliberate_camera()_** function defined in the "Caliberate Camera" section (10th cell) of the IPython notebook.  

* Calibration is done using a set of standard chessboard images that are available in the 'camera_cal/' folder.
* Object points are chosen as the corners of the 9x6 chessboard.  The chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  The object points will be the array of coordinates of the chessboard corners.
* Each of the calibration chessboard images are grayscaled and fed into OpenCV's _findChessboardCorners()_ function which returns the corner positions identified in the image.
* If corners were successfully identified, `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.
* `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time there is a successful corner detection in a test image.    
* A diagram showing the identified corners is pasted below for reference
* The `objpoints` and `imgpoints` are used to compute the camera calibration and distortion coefficients using OpenCV's  _calibrateCamera()_ function.
* The distortion coefficients obtained in the previous step are applied to correct the test image provided using OpenCV's' _undistort()_ function.  The resulting corrected image is shown below.

Chessboard with corners found  <img src="./camera_cal/corners_found9.jpg" width="600">
Distorted and Undistorted images <img src="./camera_cal/calibration1.jpg" width="300"> <img src="./camera_cal/test_undist.jpg" width="300">
 
  
   
   
### *Pipeline (single images)*

**1. Provide an example of a distortion-corrected image.**

The code for this step is contained in the "Undistortion Correction" section (14th cell) of the IPython notebook.  A sample image is read and undistorted using the coefficients calculated during camera caliberation.

![alt text][image1]

**2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.**

The code for this step is contained in the **_get_binary_warp()_** function defined in the "Extract warped binary image of the lanes" section (4th cell) of the IPython notebook.

Below are the steps taken to get a thresholded binary image :

* The function takes in a 3 channel RGB olor image and returns a warped binary image with the lane markings.
* The input image is first undistorted using the coefficients returned by camera caliberation.
* The lighting conditions are determined by calculating the average luminosity of the bottom half.  A low value would indicate shady conditions or cloudy conditions.  Refer function  **_check_lighting()_** under "Define support functions" section (3rd cell).
* If the lighting conditions are poor, then the luminosity is augmented (similar to turning on 'Head Lights' during manual driving).  This is to correct the input image before feeding it into the pipeline.  Refer function  **_switch_on_headlights()_** under "Define support functions" section (3rd cell).
* Image is converted to grayscale for gradient thresholding.  Low intensity pixels are filtered out of the gray image to remove all black pixels from image.
* Color threshold images on S and L channels of HLS color space are obtained.  Pixels present in both S and L threshold images are filtered out to form the final color threshold image. 
* Gradient thresholds images on x and y directions are obtained.  Pixels present in both x and y direction threshold images are filtered out to form the final gradient threshold image. 
* Combine the color and gradient thresholded images to form the final binary image.
* Apply mask on the binary image to filter out only the region of interest (lanes)

Below are images of the input, output and some of the intermediate images used in the processing.
![alt text][image2]

**3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.**

The code for calculating perspective M and Inverse perspective Minv is contained in the **_compute_perspective()_** function defined in the "Compute Perspective Transform" section (10th cell) of the IPython notebook.
The code for invoking warp using perspective M is contained in the **_get_binary_warp()_** function defined in the "Extract warped binary image of the lanes" section (4th cell) of the IPython notebook.

* Hard-coding approach for source and destination points has been taken for this project.
* Different perspective for different terrain types (Freeway in Project video, Highway in Challenge video and Mountain roads in Harder challenge video)
* The perspective transform M and inverse perspective transform Minv are calcuated using the source and destination points based on the terrain types
* In the **_get_binary_warp()_** function, the final masked binary threshold image is warped using the perspective transform M.

The perspective transform was verified on a test image and the results are shown below.

![alt text][image3]


**4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial**

The code for getting the left and right lane fits is contained in the **_get_line_fits()_** function defined in the "Extract left and right lane fits from a warped binary image" section (7th cell) of the IPython notebook.

Below is the algorithm used to get the left and right lane fits from a given binary thresholded warped image of the lanes :

1. Check if fresh lanes have to be identified.  This will be needed for the very first frame and also when there have been a few consequtive bad lanes
    1. If fresh lanes detection is needed, calculate fresh left and right lane pixels using using the sliding window technique.
    2. If fresh lanes detection is not needed, use the left and right fit from the previous good frame and calculate the x & y indices for the current frame
2. If there are 0 pixels are identified, then use the previous frame pixels.  This can happen in low-light conditions like shades, underpass etc.
3. Find the new left and right lane fits for the current frame from the pixels identified.
4. Sanity check is done on the identified lane (like lane width, curvature etc).  The function **_sanity_check_lanes()_** is used to perform these checks.
    1. If the sanity checks pass, then the current frame is deemed good and a new line object with the current lane parameters is created.
    2. If the line is good, then the co-efficient values are smoothed over the last 5 frames for smoothing purposes.
    3. If the sanity checks fail, then the current frame is deemed bad and the previous frame parameters are set for the current frame line object.
7. The line object is added to a list (detected_lines[]) so that it can be used in subsequent frames for smoothing

Below are the sanity check rules that have been applied :

This resulted in the following source and destination points:

| Sanity Type       | Check Applied                                                                     |  
|:-----------------:|:---------------------------------------------------------------------------------:| 
| Same Frame Check  | Minimum threshold for left and right lane Pixels Identified                       | 
| Same Frame Check  | Verify Lane widths at base and top are reasonable                                 | 
| Same Frame Check  | Verify that left and right lane curvatures are within limits                      | 
| Same Frame Check  | Verify that left and right lane curvatures are similar                            | 
| Same Frame Check  | Verify that left and right fit 2nd order co-efficients are similar                | 
| Same Frame Check  | Verify that left and right fit 1st order co-efficients are similar                | 
| Prev Frame Check  | Verify that left and right base x values are similar to values from prev frame    |
| Prev Frame Check  | Verify that left and right curvature values are similar to values from prev frame |
| Prev Frame Check  | Verify that offset from center value is similar to value from prev frame          |

Below image shows the left and right lane pixels identified in red and blue respectively:

![alt text][image4]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in lines # through # in my code in `my_other_file.py`

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in lines # through # in my code in `yet_another_file.py` in the function `map_lane()`.  Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  


```python
src = np.float32(
    [[(img_size[0] / 2) - 55, img_size[1] / 2 + 100],
    [((img_size[0] / 6) - 10), img_size[1]],
    [(img_size[0] * 5 / 6) + 60, img_size[1]],
    [(img_size[0] / 2 + 55), img_size[1] / 2 + 100]])
dst = np.float32(
    [[(img_size[0] / 4), 0],
    [(img_size[0] / 4), img_size[1]],
    [(img_size[0] * 3 / 4), img_size[1]],
    [(img_size[0] * 3 / 4), 0]])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 585, 460      | 320, 0        | 
| 203, 720      | 320, 720      |
| 1127, 720     | 960, 720      |
| 695, 460      | 960, 0        |
