# CarND-Advanced-Lane-Lines # 
Self-Driving Car Engineer Nanodegree Program

---

## Introduction ##

This is my solution to the Advanced Lane Lines project of the Udacity Self-Driving Car Engineer Nanodegree Program. The goal for the project is to write a software pipeline to identify the lane boundaries in a video from a front-facing camera on a car.

Camera calibration, color transforms, gradients, perspective transform are used in my solution. Moverover, lane curvature and vehicle position are estimated based on numerical methods.

[Here](./examples/result2.png) is an example of output images. All final output images can be seen in the folder "./output_images".

[Here](./project_video_output.mp4) is the final output video.

[Here](./writeup.pdf) is a detailed writeup report.

---

## Implementation ##


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

[image1]: ./examples/undistort_output.jpg "calibration1_undistort"
[image2]: ./examples/test1_undistort.png "test1_undistort"
[image3]: ./examples/binary_combo_example.png "Binary Example"
[image4]: ./examples/perspective_transform.png "Warp Example"
[image5]: ./examples/binary_warpped.jpg "Warped Binary"
[image6]: ./examples/FittedLine.png "FittedLine"
[image7]: ./examples/curvature_formula.png "Curvature Formula"
[image8]: ./examples/straight_lines2.jpg "Output example"
[image9]: ./examples/test2.jpg "Output example"

[video1]: ./project_video.mp4 "Video"

### Camera Calibration and Undistortion ###
My camera calibration starts by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, "object points" is just a replicated array of coordinates. "Image points" will be the (x, y) pixel position of each of the corners in the image plane. Every time I successfully detect all chessboard corners in a test image, those "object points" and "image points" will be appended into two arrays, respectively. In practice, I use `cv2.findChessboardCorners()` function in OpenCV to get all corners on grayscale of all calibration images. These corners are image points. Then I set the coordinate frame on the chessboard and get all object points corresponding to image points.

After that, I use `cv2.calibrateCamera()` function in `OpenCV` to get coefficients of matrixes distortion and calibration.
As an example, I use these matrixes to undistort "./camera_cal/calibration1.jpg". Here is the result.

![calibration1_undistort][image1]

Here is another example, where I use these matrixes to undistort "./test_images/test1.jpg". Here is the result.

![test1_undistort][image2]

### Image Pipeline ###

#### Get Theresholded Binary Images ####

After many trials, I choose to use the combination of y gradient on G channel of GRB image and S channel of HLS image to mask those undistorted images. The theresholds are showed below.

|                         | Max thereshold | Max thereshold |
|:-----------------------:|:--------------:|:--------------:| 
| Y gradient of G channel | 100            | 20             |
| S channel value         | 255            | 170            |

Here is example of apply theresholding. The source image is "./test_images/test5.jpg".

![Binary Example][image3]

#### Wrap Theresholded Image (Perspective Transform)####

I use "./test_images/straight_lines1.jpg" to get the M matrix of perspective transform, which is used in processing all images.

I manually select 4 points in the image as source points and set four distination points, and perform the perspective transform by using `cv2.getPerspectiveTransform()` function to get the warped image.
Then I slightly tune the coordinates of 4 souce points to make the lines in the warped image nearly vertical and straight.
Here is a result of perspective tranform.

![Warp Example][image4]

Then I used the M matrix to warp those thresholded binary images. Here is an example, and the source image is "./test_images/test5.jpg". 

![Warp Example][image5]

#### Detect Lane Pixels ####

_Get the histogram and find the x values of left and right line on the bottom of image_

By summing each column of the bottom half of binary wrapped image gotten from the image processing, I get the histogram of the bottom half of the image. Then from the right half and left half of the image, I choose the x value of the column with the highest value in histogram as the x-position of bases of left and right line.
Then I decide whether the right and left lines are detected from the former frame by comparing the base x values of current and former frame. If the input is a single image, then the lines in it are always not detected from the former frame.

_Find useful pixels by sliding 9 windows_

If lines in current frame is not detected in the former frame, I use sliding windows to find useful pixels.
The image is divided into left and right part evenly. For each part, 9 longitudinally connected windows with margin of 50 are used to find nonzero pixels. For each part, from the bottom of the image, the first window is centered at the x-position of bases of the line, and following windows are centered at the center of previous window’s center unless the previous window find more than 50 pixels, in which case the following window is centered at the mean positions of those pixels found in the previous window.

_Find useful pixels by searching around the fitted lines in the previous frame_

If lines in current frame has been detected in the previous frame, I just simply searching the area around the fitted lines in the previous frame with the margin of 50.

For every single test image, the pipeline uses sliding windows to find useful pixels.
Here is an example of results. Useful pixels are colored. The fitted lines in the image are second order polynomial to these pixels. The source image is ‘test5.jpg’.

![Fitted Lines][image6]

#### Further Process Lane Pixels ####

Here I define a class, Line () to calculate lane curvatures, the vehicle's position on the current lane, and filter out noise.

_Get fitted results, curvatures and the position of the vehicle_

In the class, I define a function `UpDate()` to do all the calculations.
The function takes in positions of useful pixels and use them to get coefficients of second order polynomial. According to these coefficients, the curvature can be calculated based on the following formula:

![Curvature Equation][image7]

The position of the vehicle with respect to center L can be calculated by applying:

𝐿 = ((𝑥_𝑙𝑒𝑓𝑡+𝑥_𝑟𝑖𝑔ℎ𝑡 – 1280) ∗ 𝑥𝑚_𝑝𝑒𝑟_𝑝𝑖𝑥

Where x is the x value on fitted lines when set y value as the image height (719), 𝑥𝑚_𝑝𝑒𝑟_𝑝𝑖𝑥 is the factor to convert pixel to meter.

_Filter noise_

The function `UpDate()` also stores the fitting coefficients used in the last 5 iterations. And if the new fitting coefficients of current frame have mild difference with the average coefficients of the last 5 iterations, the current frame will be considered as valid frame.
In valid frames, new fitted coefficients are used for finding and displaying lanes, calculate curvatures and vehicle position. In other frames, the average coefficients of the last 5 iterations are used to do these things. And only fitted coefficients in valid frames are stored.
If the stored set of fitted coefficients is less than a minimum number, which I set 5, the stored data will be considered to be weak, and the current frame will be consider as valid frame.

