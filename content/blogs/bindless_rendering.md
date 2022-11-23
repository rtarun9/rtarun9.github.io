---
title: "Bindless Rendering in DirectX12 and SM6.6"
draft: false
comments: true
date: 2022-09-30
weight: 1   
showtoc: true
tocopen: true
tags: ["C++", "DirectX12", "Bindless Rendering"]
cover:
    image: "/images/DX12-logo.png"
description: "Basic overview of Modern Bindless Rendering, with a focus on DX12 / HLSL."
summary: "Basic overview of Modern Bindless Rendering, with a focus on DX12 / HLSL."
---


## Why go bindless ?
If you have used the traditional binding model, you are probably familiar with the pain of setting up your root parameters, calling `ID3D12GraphicsCommandList::SetGraphicsRootDescriptorTable`or `ID3D12GraphicsCommandList::SetGraphicsRootConstantBufferView` and end up setting the wrong root parameter index and such. Modifying your bound resources even a little can cause a lot of hassle. In Bindless rendering, however, we can bind resource view's with just an index, and place all of these indices in a RootConstant (i.e a group of constants that can be bound directly to the shader. Moreover, you could use the same root signature for all of your pipelines (Graphics and Compute) :heart_eyes:.


## Using SM6.6's ResourceDescriptorHeap / SamplerDescriptorHeap
From SM6.6 onwards, we can directly index into the Cbv_Srv_Uav and Sampler heaps using just an index to access a particular resource (i.e, index into any shader visible descriptor heap). As CBV, SRV, and UAV descriptors all belong to the same descriptor heap, managing descriptors becomes very convenient and simple.
To access a resource, you can do:  
```cpp
StructuredBuffer<float3> positionBuffer = ResourceDescriptorHeap[position_buffer_index];
float3 vertexPosition = positionBuffer[vertexID]; // Here, vertexID is set using `uint vertexID: SV_VertexID'.

Texture2D<float4> texture = ResourceDescriptorHeap[texture_index];
```
You can also skip having to use Vertex Buffer's and instead use StructureBuffer's for vertex attributes, by using ResourceDescriptorHeap + the SV_VertexID semantic to get data for the current vertex shader invocation  :sparkles:. \
**Keep in mind that you still need to bind your Index Buffer manually using ```D3D12GraphicsCommandList::IASetIndexBuffer``` method!**.
  
  

You will also need to specify additional flags in your root signature (`D3D12_ROOT_SIGNATURE_FLAG_CBV_SRV_UAV_HEAP_DIRECTLY_INDEXED` and `D3D12_ROOT_SIGNATURE_FLAG_SAMPLER_HEAP_DIRECTLY_INDEXED`) to enable this functionality. There is now no need to bind DescriptorRange's in the root signature!
> Note:exclamation: : Make sure that you create your Cbv_Srv_Uav and sampler descriptor heaps with the D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE flag, and set the ShaderVisibility flag D3D12_SHADER_VISIBILITY_ALL for having a single root signature for all pipelines!

Obviously, as you are indexing into shader visible descriptor heaps, you also need to set the heaps using the `ID3D12GraphicsCommandList::SetDescriptorHeaps` call. Be sure to do this for each command list!


## Practical Example
In my renderer [Helios](https://github.com/rtarun9/Helios), I have a Buffer/Texture struct, which stores several descriptor indices (one for SRV, UAV, CBV, etc).


```cpp
struct Buffer
{  
    /* .. some functions / variables .. */
    uint32_t srvIndex{};
    uint32_t uavIndex{};
    uint32_t cbvIndex{};
};


struct Texture
{
    /* .. some functions / variables .. */
    uint32_t srvIndex{};
    uint32_t uavIndex{};
};
```


You could modify your DescriptorHeap abstraction to keep track of the current index, or you could manually do:
```cpp
uint32_t Descriptor::GetDescriptorIndex(const DescriptorHandle& descriptorHandle) const
{
        return static_cast<uint32_t>((descriptorHandle.gpuDescriptorHandle.ptr - mDescriptorHandleFromStart.gpuDescriptorHandle.ptr) / mDescriptorSize);
}
```


Now, how exactly do we go about using bindless rendering?
Say you want to render a SkyBox. You would need a few resources for this: A position buffer (in this case I am using Vertex Pulling and not Vertex Buffer), a texture, and a view projection matrix (which I store in a SceneBuffer). The complete HLSL shader code for this can be found [here](https://github.com/rtarun9/Helios/blob/master/Shaders/SkyBox/SkyBox.hlsl):
```cpp
struct SkyBoxRenderResources
{
    uint positionBufferIndex;
    uint sceneBufferIndex;
    uint textureIndex;
};


ConstantBufferStruct SceneBuffer
{
    /* .. Other stuff .. */
    float4x4 viewProjectionMatrix;
};
```

The Vertex and Pixel shader uses `ResourceDescriptorHeap` in this way :
```cpp
ConstantBuffer<SkyBoxRenderResources> renderResource : register(b0);

[RootSignature(BindlessRootSignature)]
VSOutput VsMain(uint vertexID : SV_VertexID)
{
    StructuredBuffer<float3> positionBuffer = ResourceDescriptorHeap[renderResource.positionBufferIndex];
    ConstantBuffer<SceneBuffer> sceneBuffer = ResourceDescriptorHeap[renderResource.sceneBufferIndex];

    VSOutput output;
    output.position = mul(float4(positionBuffer[vertexID], 0.0f), sceneBuffer.viewProjectionMatrix);
    output.modelSpacePosition = float4(positionBuffer[vertexID].xyz, 0.0f);
    output.position = output.position.xyww;

    return output;
}

float4 PsMain(VSOutput input) : SV_Target
{
    TextureCube environmentTexture = ResourceDescriptorHeap[renderResource.textureIndex];
    float3 samplingVector = normalize(input.modelSpacePosition.xyz);

    return environmentTexture.Sample(linearWrapSampler, samplingVector);
}
```
Note how all the indices are stored in a single constant buffer (called SkyBoxRenderResources). As this is a bunch of 32-bit root constants, setting them up on the C++ side is pretty easy:
```cpp
// gfx::X::GetSrv/CbvIndex basically checks if the buffer is null : If so, it returns -1 (or 0XFFFF'FFFF). In the HLSL side, we can check if these values are -1 or not.
// If they are, for debugging you can check return an arbitrary value such as float3(0.0f, 0.0f, 0.0f) and such.
SkyBoxRenderResources skyBoxRenderResources
{
    .positionBufferIndex = gfx::Buffer::GetSrvIndex(mPositionBuffer.get()),
    .sceneBufferIndex = gfx::Buffer::GetCbvIndex(mSceneBuffer.get()),
    .textureIndex = gfx::Texture::GetSrvIndex(mSkyBoxTexture.get());
};

// Don't forget to set your descriptor heaps!
std::array<ID3D12DescriptorHeap* const, 2u> descriptorHeaps
{ 
    mCbvSrvUavDescriptorHeap.GetDescriptorHeap(),
    mSamplerDescriptorHeap.GetDescriptorHeap()
};

commandList->SetDescriptorHeaps(static_cast<uint32_t>(descriptorHeaps.size()), descriptorHeaps.data());

// Note:exclamation:: You must set the Graphics / Compute root signature only *after* setting your descriptor heaps, as the correct heap pointers must be available when root signature is set.
commandList->SetGraphicsRootSignature(mRootSignature.Get());

// Yup, setting up these constants is that easy :heart_eyes:
commandList->SetGraphicsRoot32BitConstants(0u, 3u, reinterpret_cast<void*>(&skyBoxRenderResources), 0u);
```


## Debugging a Bindless Renderer
This is where things get just a little inconvenient compared to the traditional binding model :expressionless:.

Unlike the traditional slot-based binding model, you can't directly check the resources bound to the pipeline (buffers/textures). Instead, you can only see the indices you have setup.


- Bindless debugging in RenderDoc:
![](/images/RenderDocRenderResources.png)
I've personally found that in RenderDoc, if you know the name of the resources being bound, you could directly check it out in the 'Resource Inspector Tab'. 
![](/images/RenderDocBufferView.png)
<br>
- Bindless debugging in Nsight:
![](/images/NsightHeapEntry.png)
In Nsight, you can view the contents of your descriptor heap along with their indices in the 'Descriptor Heap view'), making debugging convenient.

On a side note, if you ever find your application running correctly with / without the visual studio debugger but get a blank render target output while running through RenderDoc / Nsight, have a check if you set the descriptor heaps *before* settings your root signature!

## Performance Considerations
> Note:exclamation:: If performance is critical, profile your code using various techniques to see what is optimal.

Bindless works great (and could be more performant, too) for most resource types, except *ConstantBuffers*, as some architectures have specific paths for them, and using bindless rendering for CBVs may not be compatible with them. Also, binding root Constant Buffer Views is fast in terms of CPU cost. 

If you are targeting post - turing Nvidia hardware, however, this may not be an issue. 

One alternative is to have a different root signature for each pipeline, where you have a RenderResources struct for your SRV / UAV / Sampler's descriptor heap indices, and use `Inline Constant Buffer Root Descriptors` for your constant buffers. For convenience, you could use Shader Reflection to make the process of setting up root parameters easy and automated.

In my new renderer, [NetherEngine](https://github.com/rtarun9/NetherEngine/), this is exactly what I do, using the DXC C++ Api for shader compilation and reflection:


```cpp
Microsoft::WRL::ComPtr<ID3D12ShaderReflection> shaderReflection{};
mUtils->CreateReflection(&reflectionBuffer, IID_PPV_ARGS(&shaderReflection));
D3D12_SHADER_DESC shaderDesc{};
shaderReflection->GetDesc(&shaderDesc);

// Get root parameters from shader reflection data.
outShaderReflection.rootParameters.reserve(shaderDesc.BoundResources);
for (uint32_t i : std::views::iota(0u, shaderDesc.BoundResources))
{
    D3D12_SHADER_INPUT_BIND_DESC shaderInputBindDesc{};
    ThrowIfFailed(shaderReflection->GetResourceBindingDesc(i, &shaderInputBindDesc));

    // A map of wstrings to uint32_t's : So that you don't need to manually set root parameter index in the ID3D12GraphicsCommandList::SetGraphicsRootConstantBufferView call.```
    outShaderReflection.rootParameterMap[StringToWString(shaderInputBindDesc.Name)] = i;

    ID3D12ShaderReflectionConstantBuffer* shaderReflectionConstantBuffer = shaderReflection->GetConstantBufferByIndex(i);
    D3D12_SHADER_BUFFER_DESC constantBufferDesc{};
    shaderReflectionConstantBuffer->GetDesc(&constantBufferDesc);

    // Each shader will have a RenderResources struct at register b0.
    if (shaderInputBindDesc.BindPoint == 0u)
    {
        const D3D12_ROOT_PARAMETER1 rootParameter
        {
            .ParameterType = D3D12_ROOT_PARAMETER_TYPE_32BIT_CONSTANTS,
            .Constants
            {
                .ShaderRegister = shaderInputBindDesc.BindPoint,
                .RegisterSpace = shaderInputBindDesc.Space,
                .Num32BitValues = constantBufferDesc.Size / 8
            }
        };

        outShaderReflection.rootParameters.emplace_back(rootParameter);
    }
    else if (shaderInputBindDesc.Type == D3D_SIT_CBUFFER && shaderInputBindDesc.BindPoint != 0)
    {
        const D3D12_ROOT_PARAMETER1 rootParameter
        {
            .ParameterType = D3D12_ROOT_PARAMETER_TYPE_CBV,
            .Descriptor
            {
                .ShaderRegister = shaderInputBindDesc.BindPoint,
                .RegisterSpace = shaderInputBindDesc.Space,
                .Flags = D3D12_ROOT_DESCRIPTOR_FLAG_DATA_STATIC
            }
        };

        outShaderReflection.rootParameters.emplace_back(rootParameter);
    }
}

// Now, in some other file where the RS and PSO are being created....

// Create root signature.
const D3D12_VERSIONED_ROOT_SIGNATURE_DESC rootSignatureDesc
{
    .Version = D3D_ROOT_SIGNATURE_VERSION_1_1,
	.Desc_1_1
	{
        .NumParameters = static_cast<uint32_t>(shaderReflection.rootParameters.size()),
		.pParameters = shaderReflection.rootParameters.data(),
		.NumStaticSamplers = 0u,
		.Flags = D3D12_ROOT_SIGNATURE_FLAG_CBV_SRV_UAV_HEAP_DIRECTLY_INDEXED | D3D12_ROOT_SIGNATURE_FLAG_SAMPLER_HEAP_DIRECTLY_INDEXED
    }
};
```

Binding your resources in this model boils down to:
```cpp
commandList->SetGraphicsRoot32BitConstants(mShaderReflection.rootParameterMap[L"renderResources"], 2u, &renderResources, 0u);
commandList->SetGraphicsRootConstantBufferView(mShaderReflection.rootParameterMap[L"transformBuffer"], mTransformBuffer.resource->GetGPUVirtualAddress());
```
Not as convenient as going completely bindless, but better than the traditional 'bindfull' model :sparkling_heart:.

## Closing Thoughts
Bindless Rendering in my opinion is great for simplifying the complex binding model of DX12, while also making changes to shaders / bound resources much easier. 

For example, With `SamplerDescriptorHeap`, you don't need to worry if a particular texture requires a wrap/clamp filter and such, which could be an issue when sampling from normal maps (granted your GLTF models have specified sampler settings and you use a GLTF loader such as TinyGLTF which makes retrieving sampler information a breeze). Here is how I load samplers in my renderer [Helios](https://github.com/rtarun9/Helios/blob/fa754656255ce2b62cd3e52e7643ffc766806cb0/Engine/Source/Scene/Model.cpp#L315) using TinyGLTF.

Also, if you cannot use SM6.6, you could have a descriptor table with several descriptor ranges (a range for each resource type, such as UAVs, SRVs, etc) which you can index into. The article by Darius Bouma in the [More Detailed Resources](#more-detailed-resources) section goes more into this. 

Personally, bindless rendering is the reason I wanted to use DX12 for my previous renderers, as it makes using the API easier and less daunting.

Thank you so much for your time! Feel free to leave comments if you felt something was lacking/incorrect or your opinions on my first post! If you would like to reach out to me, head over to the [Home page](/) to find some contact links there.

## More Detailed Resources
If you want to go deeper into bindless rendering, here are some resources I have found to be really helpful: \
[Matt Pettineo's Bindless Rendering article in Ray Tracing Gems II (chapter 17)](http://www.realtimerendering.com/raytracinggems/rtg2/index.html).\
[Traverse Research -> Bindless Rendering Series by Darius Bouma](https://blog.traverseresearch.nl/bindless-rendering-setup-afeb678d77fc). \
[Wicked Engine's DevBlog on Bindless Descriptors by Turanszki J](https://wickedengine.net/2021/04/06/bindless-descriptors/). \
[Alex Tardif's post on Bindless Rendering](https://alextardif.com/Bindless.html). \
[Game Engine Series's video on Low Level Materials and incorporating SM6.6's Dynamic Resources](https://www.youtube.com/watch?v=JItxsGsc9pI). \
[Vertex Pulling Benchmarks across hardware](https://docs.google.com/spreadsheets/d/15-V9RFoxj21PoMrfOyVLAMRN0ekejt6iiimWxyThDvk/edit#gid=0). \
[Official documentation on HLSL's Dynamic Resources](https://github.com/microsoft/DirectX-Specs/blob/master/d3d/HLSL_SM_6_6_DynamicResources.md).
