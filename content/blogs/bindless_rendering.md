---
title: "Bindless Rendering in DX12 and SM6.6"
draft: false
comments: true
date: 2022-09-30
cover:
    image: "/images/DX12-logo.png"
---


## Why go bindless ?
If you have used the traditional binding model, you are probably familiar with the pain of setting up your root parameters, calling `ID3D12GraphicsCommandList::SetGraphicsRootDescriptorTable`or `ID3D12GraphicsCommandList::SetGraphicsRootConstantBufferView` and end up setting the wrong root parameter index and such. Basically, modifying your bound resources even a little can cause a lot of hassle. In Bindless rendering, however, we can bind resources with just an index, and place all of these indices in a RootConstant (i.e a group of constants that can be bound directly to the shader :heart_eyes:. Moreover, you could use the same root signature for all of your pipelines (Graphics and Compute :smile:).


## Using SM6.6's ResourceDescriptorHeap / SamplerDescriptorHeap
From SM6.6 onwards, we can create resources from descriptors by directly indexing into the Cbv_Srv_Uav and Sampler heaps using just an index.
For example,
```cpp
    Texture2D<float4> texture = ResourceDescriptorHeap[texture_index];
    StructuredBuffer<float3> positionBuffer = ResourceDescriptorHeap[position_buffer_index];
```
You will also need to specify additional flags in your root signature (`D3D12_ROOT_SIGNATURE_FLAG_CBV_SRV_UAV_HEAP_DIRECTLY_INDEXED` and `D3D12_ROOT_SIGNATURE_FLAG_SAMPLER_HEAP_DIRECTLY_INDEXED`) to enable this functionality.


Obviously, since you are indexing into descriptor heaps, you also need to set the heaps using the `ID3D12GraphicsCommandList::SetDescriptorHeaps` call. Be sure to do this for each command list!


##Practical usage
In my renderer [Helios](https://github.com/rtarun9/Helios), I have a Buffer/Texture struct, which stores an index (one for SRV, UAV, CBV, etc).


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
    uint32_t dsvIndex{};
    uint32_t rtvIndex{};
};
```


You could modify your DescriptorHeap abstraction to keep track of the current index, or you could manually do :
```cpp
uint32_t Descriptor::GetDescriptorIndex(const DescriptorHandle& descriptorHandle) const
{
        return static_cast<uint32_t>((descriptorHandle.gpuDescriptorHandle.ptr - mDescriptorHandleFromStart.gpuDescriptorHandle.ptr) / mDescriptorSize);
}
```


Now, how exactly do we go about using bindless rendering ?
Say you want to render a SkyBox. You would need a few resources for this : A position buffer (in this case I am using Vertex Pulling and not Vertex Buffer), a texture, and a view projection matrix (which I store in a SceneBuffer). The HLSL code for this can be found [here](https://github.com/rtarun9/Helios/blob/master/Shaders/SkyBox/SkyBox.hlsl), and in a nutshell is :
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
Not how all the indices are stored in a single constant buffer (called SkyBoxRenderResources). As this is a bunch of 32-bit root constants, setting them up on the C++ side is pretty easy :
```cpp


// gfx::X::GetSrv/CbvIndex basically checks if the buffer is null : If so, it returns -1 (or 0XFFFF'FFFF). In the HLSL side, we can check if these values are -1 or not.
// If they are, for debugging you can check return an arbitrary value such as float3(0.0f, 0.0f, 0.0f), etc.
SkyBoxRenderResources skyBoxRenderResources
{
    .positionBufferIndex = gfx::Buffer::GetSrvIndex(mPositionBuffer.get()),
    .sceneBufferIndex = gfx::Buffer::GetCbvIndex(mSceneBuffer.get()),
    .textureIndex = gfx::Texture::GetSrvIndex(mSkyBoxTexture.get());
};


// Yup, setting up these constants is that easy :heart_eyes:
mCommandList->SetGraphicsRoot32BitConstants(0u, 64u, reinterpret_cast<void*>(&skyBoxRenderResources), 0u);
```


## Debugging a Bindless Renderer
This is where things get just a little inconvenient :expressionless:.
Unlike the traditional slot based binding model, you cant directly check the resources bound to the pipeline (buffers / textures). Instead, you can only see the indices you have setup.


> Bindless debugging :
![](/images/RenderDocRenderResources.png)
I've personally found that in RenderDoc, if you know the name of the resources being bound, you could directly check it out in the 'Resource Inspector Tab'. In NSight, however, you can view the contents of your descriptor heap along with their indices (in the 'Descriptor Heap view'), making debugging pretty convenient.


> Nsight Heap Entry view (HeapEntry corresponds to the index of view in the descriptor heap).
![](/images/NsightHeapEntry.png)


## Performance Considerations
(Note : If performance is critical, you should probably profile your code using various techniques to see what is optimal).


Bindless works great (and could be more performant, too) for most resource types. except *ConstantBuffers*, as there are constants for the *entire draw call*.
One alternative is to have a different root signature for each pipeline : Where you have a RenderResources struct for your SRV / UAV / Sampler's descriptor heap indices, and use `Inline Constant Buffer Root Descriptor's` for your constant buffer's. For convenience, you could use Shader Reflection to make the process of setting up root parameters easy and automated.
In my new renderer, [NetherEngine](https://github.com/rtarun9/NetherEngine/), this is exactly what I do, using the DXC C++ Api for shader compilation and reflection. :


```cpp
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
```


Binding your resources in this model boils down to :
```cpp
commandList->SetGraphicsRoot32BitConstants(mShaderReflection.rootParameterMap[L"renderResources"], 2u, &renderResources, 0u);
commandList->SetGraphicsRootConstantBufferView(mShaderReflection.rootParameterMap[L"transformBuffer"], mTransformBuffer.resource->GetGPUVirtualAddress());
```
Not as convenient as going completely bindless, but better than the traditional 'bindfull' model :sparkling_heart:.


## More detailed resources
If you want to go deeper into bindless rendering, here are some resources I have found to be really helpful : \
[Traverse Research -> Bindless Rendering Series by Darius Bouma](https://blog.traverseresearch.nl/bindless-rendering-setup-afeb678d77fc). \
[Wicked Engine's DevBlog on Bindless Descriptor's by Turanszki J](https://wickedengine.net/2021/04/06/bindless-descriptors/). \
[Alex Tardif's post on Bindless Rendering](https://alextardif.com/Bindless.html).

