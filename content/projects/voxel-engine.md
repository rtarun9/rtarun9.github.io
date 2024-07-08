---
title: "voxel-engine [C++ DirectX12 voxel engine]"
draft: false
weight: 3
cover:
    image: "/images/voxel-engine/thumbnail-wireframe.png"

description: "A simple voxel engine made using DirectX12 and C++"
summary: "A simple voxel engine made using DirectX12 and C++"
---
# voxel-engine: C++ / DirectX12
> Github repo : https://github.com/rtarun9/voxel-engine
###  A simple voxel engine made using DirectX12 and C++

# Showcase Video
{{< youtube E0T0UMnOggg >}}


## Features
* Implemented advanced GPU optimization techniques, including GPU Culling and Indirect Rendering,
achieving smooth rendering of large scenes with over 100 million vertices at 60 FPS.
* Integrated a multi-threaded chunk loading system with Async Copy Queue support to make rendering
and GPU copy operations independent of each other.
* Implemented Reverse Z to mitigate depth buffer precision issues in large scenes, evenly distributing depth buffer precision, and enabling infinite far planes
