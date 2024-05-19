---
title: "Stereo Depth Estimation [Python OpenCV2 | C++ DirectX11]"
draft: false
weight: 6
cover:
    image: "/images/StereoDepthDisparity.png"
description: "Estimation of depth map given stereo images and camera calibration parameters."
summary: "Estimation of depth map given stereo images and camera calibration parameters."
---

# Stereo Depth Estimation : Python/OpenCV2 C++/DirectX11 
> Colab Link : https://colab.research.google.com/drive/1Ey4mfWyh3wSpEplk6TY8eJepNdbtIoJn?usp=sharing 

# Showcase Video Of Point Cloud Visualizer
{{< youtube rrfGdJUEJxk >}}

### [Note : This was a group project of 5. The techniques noted below are my personal contributions towards the project]
## Features
* Implemented Morphological transforms such as Hit or Miss transform, Erosion, Dilation, Opening, Closing,
Template Matching.
* Added several edge detection algorithms such as Sorbel edge detectors, Laplacian edge detectors, and Unsharp
masking.
* Disparity/Depth map generation by extracting relevant camera calibration data.
* Implemented segmentation algorithms such as WaterShed segmentation and Colour space segmentation (HSV).
* For pre-processing of images, implemented frequency domain filters such as Butterworth’s low/high pass filter,
Ideal low/high pass filter
* Created a C++/DirectX11 application to represent the depth map in form of a point cloud visualizer.


### Spatial Domain Sharpening Filters

> Source / Original Image
![](/images/spatial_sharpened_image/source.png)
> N4 Laplacian Mask / Sharpened Image
![](/images/spatial_sharpened_image/n4laplacian.png)
![](/images/spatial_sharpened_image/n4sharpened.png)
> N8 Laplacian Mask / Sharpened Image
![](/images/spatial_sharpened_image/n8laplacian.png)
![](/images/spatial_sharpened_image/n8sharpened.png)
> Sorbel Mask / Sharpened Image
![](/images/spatial_sharpened_image/sorbelmask.png)
![](/images/spatial_sharpened_image/sorbelsharpened.png)
> Unsharp Masking Mask / Sharpened Image
![](/images/spatial_sharpened_image/unsharpmask.png)
![](/images/spatial_sharpened_image/unsharpsharpened.png)




### Frequency Domain Blurring / Sharpening Filters
> Ideal Low and High Pass Filter Masks
![](/images/frequency_blur_and_sharp/ilpf_ihpf.png)
> Application of Ideal Low and High Pass Filters
![](/images/frequency_blur_and_sharp/ilpf.png)
![](/images/frequency_blur_and_sharp/ihpf.png)
![](/images/frequency_blur_and_sharp/ihpf_2.png)

> Application of ButterWorth's Low and High Pass Filters
![](/images/frequency_blur_and_sharp/blpf.png)
![](/images/frequency_blur_and_sharp/bhpf.png)

### Other Image Processing Techniques

> Color Space Segmentation (HSV)
![](/images/image_processing/colorspace_segmentation.png)
![](/images/image_processing/colorspace_segmentation_2.png)


> Homomorphic Transformations
![](/images/image_processing/homomorphic.png)


> Morphological Closing
![](/images/image_processing/closing.png)


> Hit or Miss Transform (object to hit : The 2 wheeler Driver)
![](/images/image_processing/hit_or_miss_transform.png)


> Template Matching
![](/images/image_processing/template_matching.png)


> Disparity and Depth Map Generation
![](/images/image_processing/disparity_generation.png)
![](/images/image_processing/depth_generation.png)
