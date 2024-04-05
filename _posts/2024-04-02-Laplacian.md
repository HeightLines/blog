---
layout: post
title:  "Laplacian"
date:   2024-04-02 14:10:00 +0030
---

HeightLines uses the videos recorded by the user as the source of raw data. Using video to scan the body was chosen over images for the following reasons:

1. Convenience: To require the user to pose still is a hassle - especially for regions like the back. This process is exactly what a person would do if he was merely examining himself with the phone’s flashlight and therefore feels natural.
2. Maximize coverage: In a 5 second duration, one can capture much more surface area with a video than taking images.
3. Consistency: The exposure time in a video stays the same across the frames and there is minimal automatic adjustments by the device's firmware.

For any user recorded video, some frames will be sharper than the rest due to factors like the movement of the subject or the camera. We only want to process and store these frames. I compared intensity-variance and edge-variance (Laplacian) based metrics for this task.

__Intensity Variance__  
Variance of the pixel intensity is a simple and fast approach where you are trying to measure whether the pixels in a greyscale image have similar intensities or drastically different. Therefore, a blurry image will have lower variance than a well focused one. [This 2001 presentation](https://www.csl.cornell.edu/~cbatten/pdfs/batten-image-processing-sem-slides-scanning2001.pdf) shows the variance as a preferred measure for autofocusing in electron microscopes.

{% highlight python %}
def intensity_variance(self, gray):
    variance = np.var(gray)
    return variance
{% endhighlight python %}

Because we are dealing with casually recorded videos, the variations in the background and the subjects will be significant which is unlike a defined environment like the stage of a microscope. Therefore, instead of an absolute value, a normalized variance could be useful here, and I tried to measure it as intensity variance per edge pixel. This means we need to estimate the number of edges in a frame, and John Canny’s edge detector is a popular choice. It essentially uses the gradient orientation heuristic to detect the local intensity minima (edges). The cutoff for how intense a minima needs to be is determined by the amount of gaussian blur applied in this algorithm. The result is a 2D array with values 255 on the edge and 0 for the rest of the image. We can use the number of pixels marked as edge to get the desired metric intensity variance per edge pixel.

{% highlight python %}
def normalized_intensity_variance(self, gray: np.ndarray) -> float:
    variance = np.var(gray)
    v = np.median(gray)
    sigma = 0.33
    lower = int(max(0, (1.0 - sigma) * v))
    upper = int(min(255, (1.0 + sigma) * v))
    edges = cv2.Canny(gray, lower, upper)
    edge_pixels = np.sum(edges > 0)
    if edge_pixels == 0:
        return 0
    normalized_variance = variance / edge_pixels
    return normalized_variance
{% endhighlight python %}

__Edge Variance__  
For the edge-variance based metric I simply used the variance of the Laplacian of the greyscale image. Which means the variance of the second derivative of the intensity aka the rate of change (sum in the x and y direction) of intensity. Therefore, a high variance implies presence of strong and prominent edges.

{% highlight python %}
def laplacian_variance(self, gray):
    return cv2.Laplacian(gray, cv2.CV_64F).var()
{% endhighlight python %}

__Comparision__  
To compare these metrics a simple test video can be used which has clear segments of vigorous  movement and calm recording. Towards the end of the test video the palm is turned twice and briefly held stable after each turn.

![intensity_metric](/blog/assets/images/intensity.png)

The intensity variance metric (__Dict 2__) captures these segments but is prone to noise likely changes in brightness of the environment. This can be observed in the initial 100 frames in the graph above where the subject’s hand and the camera are in rapid motion. On examining the images corresponding to the highest and lowest values I found that intensity variance is not picking ideal candidates in our context. There isn’t much remarkable for the normalized intensity variance curve (__Dict 3__) - it also appears to suffer from the brightness induced noise.

![laplacian_metric](/blog/assets/images/laplacian.png)


The Laplacian metric (__Dict 1__) has clearly worked the best with distinct peaks for sharp frames. Unlike the previous two, it is also immune to the brightness variations. Therefore, I have chose this for filtering out the best frames to process further. Experimentally determined cutoff of __8__ is used at the moment and this approach along with adaptive sampling should suffice for now.