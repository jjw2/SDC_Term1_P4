# Advanced Lane Finding
---

[//]: # (Image References)

[image1]: ./md_imgs/chessboards.png "chessboards"
[image2]: ./md_imgs/hist.png "hist"
[image3]: ./md_imgs/hist_img.png "hist_img"
[image4]: ./md_imgs/lane_curv_ofst.png "lane_curv_ofst"
[image5]: ./md_imgs/projected_lane.png "projected_lane"
[image6]: ./md_imgs/sliding_window.png "sliding_window"
[image7]: ./md_imgs/srch_around_poly.png "srch_around_poly"
[image8]: ./md_imgs/thresholding.png "thresholding"
[image9]: ./md_imgs/thresholding1.png "thresholding1"
[image10]: ./md_imgs/undist_chessboard.png "undist_chessboard"
[image11]: ./md_imgs/undist_road.png "undist_road"
[image12]: ./md_imgs/warp_dest.png "warp_dest"
[image13]: ./md_imgs/warp_imgs.png "warp_imgs"
[image14]: ./md_imgs/warp_src.png "warp_src"

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

I will be touching on the important aspects of the [rubric](https://review.udacity.com/#!/rubrics/571/view) in this writeup.

---
## Setup
Starting materials and instructions can be found [here](https://github.com/udacity/CarND-Advanced-Lane-Lines).

## 1. Camera Calibration
Camera calibration is performed using chessboard images provided by Udacity and OpenCV libraries.

It's assumed that the chessboard is oriented on the x-y plane in the real world, such that the z coordinate of any point on the board is 0. The "corners" of the board in this case are where 4 adjacent tiles meet (2 white and 2 black), which excludes areas along the edge of the image. In this case, the "real world" coordinates of such corners are just assigned integer values (i.e.: they're not actually measured), since they are all equally spaced.

The OpenCV function "findChessboardCorners" is used to find the corners in each of the images. Images are only used in calibration if all of the  chessboard corners are found; otherwise, they are excluded.

Once points in the images have been correlated with points in the real world, their respective coordinates are passed to the OpenCV "calibrateCamera" function, which creates a matrix that can be used to undistort any images taken with the same camera using the OpenCV "undistort" function.

The set of calibration chessboard images that were used in this project are shown below.

![alt text][image1]

## 2. Distortion Correction
Using the matrix created during camera calibration, an undistort utility ("undistort" in the code) using the OpenCV "undistort" function is created. A side-by-side comparison of one of the raw and undistorted chessboard images shown below provides the best illustration of undistortion. A test image from the video is also shown for comparison, though it's a little tougher to make out the difference; it's most noticeable along the hood of the car at the bottom corners of the image.

Chessboard Images:
![alt text][image10]

Road Images:
![alt text][image11]


## 3. Thresholding
To pick out lane lines, I tested several different combinations of sobel thresholding (magnitude, direction, etc) and color thresholding (red and HLS colorspaces) as covered in the Udacity classroom sessions. However, for the project video, I found that my trusty old HSV color selection utility - developed in the first project of this course and found [here](https://github.com/jjw2/SDC_Term1_P1) - worked great on its own!

Essentially, this function converts the subject image to the HSV color space, and applies a filter on each of the 3 channels to isolate yellow and white colors. It even does a decent job of picking out the lane lines on the concrete portions of the road, which is, obviously, where finding the lane lines is the hardest.

Examples of thresholding are shown below side-by-side.

On Asphalt:
![alt text][image8]

On Concrete:
![alt text][image9]

You can see that lines and isolation quite well on both asphalt and concrete.


## 4. Perspective Transform
After thresholding, a perspective transform is performed to obtain a top-down view of the road surface for use in lane finding. The perspective transform was completed using the OpenCV "getPerspectiveTransform" and "warpPerspective" functions.

First, an image containing straight lane lines was selected - in this case, from the example images provided, since the project video didn't contain sections where the car was centered in the lane and the lane lines were straight. The image was first undistorted, and then source points for use in the perspective transform were selected by hand. The image and the source polygon are shown below.

![alt text][image14]

Next, destination points were chosen; these points map the region of the source image to be transformed (outlined by the polygon in the image above), and where that region will be located in the destination image. OpenCV getPerspectiveTransform takes source and destination points and creates a transform matrix that can be used with the warpPerspective function to transform an image; note that it can also be inverted and used to unwarp an image. The image below shows the result of the perspective transform/warp, with the destination polygon drawn.

![alt text][image12]

The image below shows the result of warping on a thresholded test image (original on the left, warped on the right).

![alt text][image13]

## 5. Finding Lane Lines
The next step of the process is finding the lane lines in the thresholded images.

First, a histogram of the pixels in each image is generated. In this case, only the bottom half of the images were used (i.e.: the half of the image closest to the car). The images below show the histogram result for one of the sample images.

![alt text][image3] ![alt text][image2]

Next, a sliding window technique was used to capture the (x,y) points that make up each lane line. The technique searches for pixels within a series of "windows" that each make up a sub-section of the image. In this case, the first sliding window is centered on x location of the two peaks in the histogram (based on the assumption that the peak in the left half of the histogram is associated with the left lane, and the peak in the right half of the histogram is associated with the right lane). Subsequent sliding windows - which move from the bottom of the image to the top with each iteration - are located based on the average x location of the pixels found in the previous window.

Once the sliding window technique has identified the pixels associated with each lane, a polynomial fitting function is used to apply a second order polynomial fit to the pixels that describes the lane line.

An image of the sliding windows and the resulting polynomials for a sample image is shown below.

![alt text][image6]

The sliding window technique works well for finding lane lines in any single image, but given that this system is finding lane lines in many successive images taken in rapid succession (i.e.: a video...), the sliding window technique - being relatively computationally expensive - doesn't have to be used on each image. The pipeline can be optimized such that, if a polynomial fit exists from a previous iteration of lane finding, a search can be performed in a region around the last known value of each lane line. Of course, this is based on the assumption that the lane lines aren't going to be moving drastically in each successive frame of the video.

The function below implements a pixel search within a window around a previous polynomial fit, and the figure below illustrates the search window (in green) the found pixels (in color) and the resulting polynomial fit for a sample image. Note that the sample image in this case is simply the same image on which the pixel search was performed above, but the intent here is simply to illustrate the process.

![alt text][image7]

## 6. Calculating Lane Curvature
Once polynomials have been fitted to lane lines, curvature can be calculated. The calc_curv function uses a standard formula for calculating curvature of 2nd order polynomials. Note however, that we care about curvature in the real world. As a result, the polynomial fits that have been calculated must be converted into real-world coordinates. The calc_curv function performs this conversion using conversion factors - the number of meters per pixel in the perspective-transformed images above - that were calculated by hand based on known quantities -> the width of the lane, and the length of the dashed right lane line.

Since curvature changes along the polynomial, the point at which curvature will be calculated must also be provided. In this case, since the calc_curv function operates in real world coordinates, it expects a value in meters, starting from the top of the perspective-transformed images. Here, I'll be using a value of 20m, which is roughly the length of the segment of road shown in the perspective-transformed images (i.e.: calculating curvature at the position of the vehicle).


## 7. Projecting Back to the Road
Now that the lane has been identified and lane lines have been found, we can project the lane back onto the road in the original image. In this case, the OpenCV fillPoly function is used to fill in the lane between the left and right lane line polynomials, and the inverted perspective transform matrix is used to project the lane back down onto the image of the road. An example image is shown below.

![alt text][image5]

## 8. Lane Position and Tracking
The functions above all handle finding lane lines in a single image. Given that we're tracking lane lines in a video, we can use information from successive images to help smooth out our lane projections/estimates and also deal with cases where we're not able to detect lane lines in a given frame. In order to track lane lines over time, classes were created to process and store information.

A Line class was created to process store information about each lane line:
- polynomial fit, with history
- curvature values
- x positions of the lane lines at the bottom of the image, with history
- status of the fit for the line (failure, etc)

A LaneInfo class was created to store information specific to the lane:
- curvature, with history
    - each curvature entry in the history is assumed to be the average of the curvatures of the two lines
- center line offset, with history
    - the offset is calculated based on the difference between the center of the image and the point in the middle of the average x values (at the bottom of the image) of the left and right lines.

Within the LaneInfo class, the values of lane curvature and center line offset are rate limited, and a maximum limit is put on curvature. This was done for a couple reasons:
1. to smooth the output values for curvature and lane offset, so that they're a little easier to read in the video (otherwise, then can jump around a lot from loop to loop, making the values difficult to read)
2. radius of curvature goes to infinity as the road straightens out; putting a cap on the radius of curvature (at 10 km in this case) also makes the values easier to read, and allows for the rate limiting above (i.e.: it would be difficult to rate limit the values if they were going to infinity, as they could "wind up," which would result in large errors when transitioning from a straight road (infinite radius of curvature) to a curved road.

See the comments in each of these classes for more information.


The proc_img function implements the full image processing pipeline and stores information in the Line and LaneInfo classes. The breakdown of the logic is as follows:
1. Undistort, threshold, and warp each image
2. If we've failed to fit either of the two lines, run the full sliding window pixel search; otherwise, search around the previous polynomial to find lane lines
    - the fit is considered "failed" if a fit hasn't been found in the last n frames of the image (with n currently being set to 5)
3. If we're able to find a new fit, do a sanity check on the fit to confirm it's good, with the sanity check consisting of:
    - a check of lane width
    - a check that the difference between successive curvatures isn't too great
4. If the sanity check passes for both lines, report a successful fit and push the appropriate values to the history buffers
5. If the sanity check fails for either line, report a failed fit instance, and don't push any values, meaning the last best values of fit, curvature, lane offset, etc, will be used for the current frame
6. Project the best fit of the lane down to the original image and print statistics on the image (curvature, center line offset)


An example image with curvature and center line offset is shown below.

![alt text][image4]

## 9. Generating the Video

The resulting video is titled "project_video_output."

## 10. Discussion

### Issues
I wouldn't say that I faced any significant issues while completing this project. However, I probably spent the most time investigating various methods for actually finding the lane lines, and, surely, in practice this must be the most challenging aspect of lane finding. The project video presents an absolutely optimal scenario for lane finding: a bright sunny day driving on dark asphalt with perfect lane lines. Performing the same task in different environments and different weather conditions would drastically increase the difficulty of the task.

### Robustness & Improvements
As mentioned above, I think the most challenging aspect of the project as a whole, and thus the area that might cause the most problems, is picking out the actual lane lines with a high level of confidence. Once the lane lines (and thus the lane lines) are found tracking them over time should be relatively straight-forward (though I'd recommend some techniques that I'm not using here to improve robustness; more below).

The first step in this process, obviously, is picking out the lane line pixels from the image. While in the project video I was able to use my trusty color-selection utility to find the lane line pixels, this solution would probably struggle in other scenarios where the lane lines were not so clearly visible/detectable - even in this case, the method used struggled slightly on the concrete sections of this video. Additional testing using various approaches in various environments would most likely yield a more robust method for finding lane line pixels across the board. It may even be the case that different filters would have different levels of performance in different scenarios (sunny vs dark vs rainy vs concrete road vs asphalt, etc), and I could imagine running several different filters and comparing results, or perhaps having a high-level decision tree that selects an appropriate filter based on an analysis of current operating conditions.

Even with the improvements noted above, it's unlikely that we'd find a method that could pick out only lane line pixels with absolute robustness in every frame of a video in every environment. Therefore, overall robustness could be improved by performing an analysis of the "confidence level" of a given pixel search. Here are some examples:
- Simplest: assign a confidence based on the number of windows in which an appropriate number of lane line pixels were found in the sliding window search
- Better: perform a statistical analysis of the distribution of found lane line pixels within a given window; this could provide information regarding how "cleanly" the lane line was picked out from the surrounding road. For example, right now, my algorithm is simply checking how many pixels were found within a given window; at the moment, however, the algorithm is not doing a check to analyze the likelihood that all of these pixels are part of the lane line. If they were all part of a line, we'd expect them to be clustered relatively close together, for example, rather than spread apart. One could use the mean and standard deviation or variance of a histogram of lane line pixels within a given window (or some other method) to assess the pixel distribution and determine the likelihood that the pixels actual form a line. The height of the windows in this case would need to be small (i.e.: not the whole image), as the curvature of the line within the window would affect pixel distribution, but in theory, this method would provide more information regarding the robustness of the pixel search.
    - If a given pixel search was found to have a low level of confidence, it could be discarded, or potentially added to the lane history queue with an appropriate weighting factor


As far as tracking goes, at the moment, this algorithm is considering the line search a failure if one or both of the lines is not found or fails the sanity check. Given an appropriate estimate of the "confidence level" of a given found lane line (tracked over time, perhaps) the system could more robustly infer the position of the opposite line from a found line. For example, if only one line is found, but the confidence level associated with that lane line is high, one may infer the position/curvature of the opposite line. However, if one line is found with a low level of confidence, it may be prudent to ignore the result. Now, while I have been conservative in only using fits where both lines are found, it would be possible to implement something like this without a confidence level per lane line (i.e.: a simple sanity check is being performed), a confidence level tracking feature would surely add robustness to such an approach.


Finally, though not possible in this project because the information is not available, this system would benefit greatly from information about the dynamics of the vehicle (ex: pitch, roll, vertical acceleration, perhaps more). You can see in the video that the fit fails or doesn't quite fit the lanes when the car is going over bumps. In some cases, for example, the sanity check fails due to the calculated width of the lane exceeding the calibrated width (set in the LaneInfo class), which is due to changes in the orientation of the camera as the vehicle goes over a bump (ex: as the shocks compress after going over a bump, the camera moves closer to the ground, making the lane appear wider than when viewed from the standard camera position). Along the same lines, changes in pitch and/or roll affect the curvature calculation.
