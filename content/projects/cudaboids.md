---
title: "Flocking Boids Simulation (Cuda C++)"
draft: false
weight: 4
cover:
    image: "/images/cuda_boids/thumbnail.png"
description: "A simple flocking boids simulation done using C++ and Cuda."
summary: "A simple flocking boids simulation done using C++ and Cuda."
---

# FLocking Boids Simulation : Cuda / C++ 
> Github  Link : [https://github.com/rtarun9/Cuda-Flocking-Boids](https://github.com/rtarun9/Cuda-Flocking-Boids)

# Showcase Video
{{< youtube Ea_GfM1cmzQ >}}

### [Note : The starting code / project template is adapted from : https://github.com/CIS565-Fall-2022/Project1-CUDA-Flocking]

## Techniques Implemented

* (i) Brute Force approach : Each boid iterates over all other boids present in the scene.
* (ii) Uniform grid: Each boid iterates over boids only in neighboring spatial grids.
* (iii) Coherent Grid: Similar to the uniform grid, but memory access is more linear (requires fewer memory indirections, which leads to massive performance gains!).

## Performance Analysis
![](/images/cuda_boids/perf_no_visualization.png)
![](/images/cuda_boids/perf_with_visualization.png)

## References
[CIS 5650 GPU Programming and Architecture](https://cis565-fall-2023.github.io/)