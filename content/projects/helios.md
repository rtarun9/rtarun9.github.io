---
title: "Helios [C++20 DirectX12 renderer]"
draft: false
weight: 1
cover:
    image: "/images/IBL1.png"
---

# Helios : C++ / DX12 Renderer 
> Github repo : https://github.com/rtarun9/Helios 
###  A Experimental DirectX12 Graphics renderer for trying out various rendering techniques.

# Showcase Video
{{< youtube hKeVVCpzVhQ >}}


## Features
* Bindless Rendering (Using SM 6.6's Resource / Sampler descriptor Heap).
* Normal Mapping.
* Diffuse and Specular IBL.
* Physically based rendering (PBR).
* Blinn-Phong Shading.
* Deferred Shading.
* HDR and Tone Mapping.
* Instanced rendering.
* OmniDirectional Shadow Mapping.
* Editor (ImGui Integration) with Logging and Content Browser.
* Compute Shader mip map generation.
* Multi-threaded asset loading.

> PBR and IBL
![](/images/IBL1.png)
![](/images/IBL2.png)
![](/images/IBL3.png)

> Omni-directional Shadow Mapping (With PCF)
![](/images/PCFShadows1.png)

> Editor (using ImGui)
![](/images/Editor1.png)

> Deferred Shading
![](/images/Deferred1.png)
