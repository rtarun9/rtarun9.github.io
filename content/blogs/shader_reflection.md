---
title: "Shader Reflection using the DirectX Shader Compiler"
draft: false
comments: true
date: 2022-11-23
showtoc: true
weight: 3
tocopen: true
tags: ["C++", "DirectX Shader Compiler", "DXC", "HLSL", "Shader Reflection"]
cover:
    image: "/images/HLSLReflection.png"
description: "Basic overview of simplifying root signature creation using the DirectX Shader Compiler for Shader Reflection and Compilation."
summary: "Basic overview of simplifying root signature creation using the DirectX Shader Compiler for Shader Reflection and Compilation."
---

> Note:exclamation:: If you are considering using Shader Reflection to simplify the root signature creation process (as well as your resource binding system), you might want to have a look at Bindless Rendering as well. I have a blog post on bindless rendering which you can find [here.](https://rtarun9.github.io/blogs/bindless_rendering/)

## What is Shader Reflection ?
Let's consider a scenario where you want to modify your shader by adding/removing resources that are bound to it (for example, separating your global constant buffer into a ‘model transform buffer’ and a ‘camera buffer’, adding a 'emissive texture' and so on).

Making these simple change, however, is not convenient, and requires quite a lot of work. First, you need to modify your shaders. Then, you need to modify your root signature on the application side to match the shaders (as the root signature defines what resources are bound to the graphics pipeline).

Notice how you make changes in both your application code and your shader code 😒. With shader reflection, you can make changes just in your shader, and automate root signature creation 😍.


## Using the DirectX Shader Compiler (DXC)
We can use [DXC (DirectX shader compiler)](https://github.com/microsoft/DirectXShaderCompiler) to compile our HLSL programs to the DirectX Intermediate Language (DXIL) representation. DXC is based on LLVM / Clang and is set to replace the old FXC compiler for HLSL. It also supports SPIR-V CodeGen! :sunglasses:

By using the DXC C++ API, we can compile our shaders and also 'reflect' data from our shaders (i.e getting some information on what resources are used by our shaders). This information can be used to create a simple system that can help greatly in specifying and creating root signatures automatically.

To use the DXC C++ API, be sure to add the following includes : 
```cpp
#include <dxcapi.h>
#include <d3d12shader.h> // Contains functions and structures useful in accessing shader information.
```

The following structures provided by DXC will be used to compile our shaders and get reflection/root signature/errors etc from the compiled shader blob :
* IDxcCompiler : Responsible for the actual compilation of shaders.
* IDxcUtils : We will use it to load the shader file to a blob.
* IDxcIncludeHandler : For custom include logic, you can create your own include handler. In most cases, using the default include handler would suffice.

Creating these core resources is fairly simple : 
```cpp
throwIfFailed(::DxcCreateInstance(CLSID_DxcUtils, IID_PPV_ARGS(&utils)));
throwIfFailed(::DxcCreateInstance(CLSID_DxcCompiler, IID_PPV_ARGS(&compiler)));
throwIfFailed(utils->CreateDefaultIncludeHandler(&includeHandler));
```

Next, you need to specify the compilation arguments to be used by DXC (these are very similar to using DXC via the command line). I will be adding the arguments to a `std::vector<LPCWSTR>` for convenience.
```cpp
std::vector<LPCWSTR> compilationArguments
{
    L"-E",
    entryPoint.c_str(),
    L"-T",
    targetProfile.c_str(),
    DXC_ARG_PACK_MATRIX_ROW_MAJOR,
    DXC_ARG_WARNINGS_ARE_ERRORS,
    DXC_ARG_ALL_RESOURCES_BOUND,
};

// Indicate that the shader should be in a debuggable state if in debug mode.
// Else, set optimization level to 3.
if constexpr (DEBUG_MODE)
{
    compilationArguments.push_back(DXC_ARG_DEBUG);
}
else
{
    compilationArguments.push_back(DXC_ARG_OPTIMIZATION_LEVEL3);
}
```

Now, you can use the IDxcUtils and IDxcCompiler to load your shader into a blob and compile it!

```cpp
// Load the shader source file to a blob.
Comptr<IDxcBlobEncoding> sourceBlob{};
throwIfFailed(utils->LoadFile(shaderPath.data(), nullptr, &sourceBlob));

DxcBuffer sourceBuffer
{
    .Ptr = sourceBlob->GetBufferPointer(),
    .Size = sourceBlob->GetBufferSize(),
    .Encoding = 0u,
};

// Compile the shader.
Microsoft::WRL::ComPtr<IDxcResult> compiledShaderBuffer{};
const HRESULT hr = compiler->Compile(&sourceBuffer,
                        compilationArguments.data(),
                        static_cast<uint32_t>(compilationArguments.size()),
                        includeHandler.Get(),   
                        IID_PPV_ARGS(&compiledShaderBuffer));
if (FAILED(hr))
{
    fatalError(std::wstring(L"Failed to compile shader with path : ") + shaderPath.data());
}

// Get compilation errors (if any).
Comptr<IDxcBlobUtf8> errors{};
throwIfFailed(compiledShaderBuffer->GetOutput(DXC_OUT_ERRORS, IID_PPV_ARGS(&errors), nullptr));
if (errors && errors->GetStringLength() > 0)
{
    const LPCSTR errorMessage = errors->GetStringPointer();
    fatalError(errorMessage);
}
```

\
Now, you can finally create your ID3D12ShaderReflection object :rocket:! \
This interface will be used to access shader information such as the input parameter descriptions (for automating input layout element description), getting constant buffer data by index or by name, etc.


```cpp
// Get shader reflection data.
Comptr<IDxcBlob> reflectionBlob{};
throwIfFailed(compiledShaderBuffer->GetOutput(DXC_OUT_REFLECTION, IID_PPV_ARGS(&reflectionBlob), nullptr));

const DxcBuffer reflectionBuffer
{
    .Ptr = reflectionBlob->GetBufferPointer(),
    .Size = reflectionBlob->GetBufferSize(),
    .Encoding = 0,
};

Comptr<ID3D12ShaderReflection> shaderReflection{};
utils->CreateReflection(&reflectionBuffer, IID_PPV_ARGS(&shaderReflection));
D3D12_SHADER_DESC shaderDesc{};
shaderReflection->GetDesc(&shaderDesc);
```

## Reflecting Input Parameters
You no longer need to manually specify a input layout for setting up your vertex buffers :fire:. This makes modifying vertex buffers very easy.
We can use the `ID3D11ShaderReflection::GetInputParameterDesc` method to do so : 
```cpp
// Setup the input assembler. Only applicable for vertex shaders.
if (shaderType == ShaderTypes::Vertex)
{
    inputElementSemanticNames.reserve(shaderDesc.InputParameters);
    inputElementDescs.reserve(shaderDesc.InputParameters);

    for (const uint32_t parameterIndex : std::views::iota(0u, shaderDesc.InputParameters))
    {
        D3D12_SIGNATURE_PARAMETER_DESC signatureParameterDesc{};
        shaderReflection->GetInputParameterDesc(parameterIndex, &signatureParameterDesc);

        // Using the semantic name provided by the signatureParameterDesc directly to the input element desc will cause the SemanticName field to have garbage values.
        // This is because the SemanticName filed is a const wchar_t*. I am using a separate std::vector<std::string> for simplicity.
        inputElementSemanticNames.emplace_back(signatureParameterDesc.SemanticName);    

        inputElementDescs.emplace_back(D3D12_INPUT_ELEMENT_DESC{
                    .SemanticName = inputElementSemanticNames.back().c_str(),
                    .SemanticIndex = signatureParameterDesc.SemanticIndex,
                    .Format = maskToFormat(signatureParameterDesc.Mask),
                    .InputSlot = 0u,
                    .AlignedByteOffset = D3D12_APPEND_ALIGNED_ELEMENT,
                    .InputSlotClass = D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 
                    // There doesn't seem to be a obvious way to 
                    // automate this currently, which might be a issue when instanced rendering is used :weary:
                    .InstanceDataStepRate = 0u,
                });
    }

    inputLayoutDesc = 
    {
        .pInputElementDescs = inputElementDescs.data(),
        .NumElements = static_cast<uint32_t>(inputElementDescs.size()),
    };
}
```

The `D3D12_SIGNATURE_PARAMTER_DESC` struct is as follows : 
```cpp
typedef struct _D3D12_SIGNATURE_PARAMETER_DESC {
  LPCSTR                      SemanticName;
  UINT                        SemanticIndex;
  UINT                        Register;
  D3D_NAME                    SystemValueType;
  D3D_REGISTER_COMPONENT_TYPE ComponentType;
  BYTE                        Mask;
  BYTE                        ReadWriteMask;
  UINT                        Stream;
  D3D_MIN_PRECISION           MinPrecision;
} D3D12_SIGNATURE_PARAMETER_DESC;
```
Most attributes are self explanatory, except `Mask`. This represents the number of components the element is using in binary form. In other words, a float3 would be represented by 7 (b'111), a float2 by 3 (b'11) and so on.

## Reflecting shader bound resources (Cbuffers, Textures, etc).
Lets get to the fun parts now :star:. Using DXC and the structs and interfaces provided by the d3d12shader.h header file, we can get data on various resources bound to the shader (constant buffers, textures, etc).

First, lets check out how to get constant buffer data.
```cpp
for (const uint32_t i : std::views::iota(0u, shaderDesc.BoundResources))
{
    D3D12_SHADER_INPUT_BIND_DESC shaderInputBindDesc{};
    throwIfFailed(shaderReflection->GetResourceBindingDesc(i, &shaderInputBindDesc));

    if (shaderInputBindDesc.Type == D3D_SIT_CBUFFER)
    {
        rootParameterIndexMap[stringToWString(shaderInputBindDesc.Name)] = static_cast<uint32_t>(rootParameters.size());
        ID3D12ShaderReflectionConstantBuffer* shaderReflectionConstantBuffer = shaderReflection->GetConstantBufferByIndex(i);
        D3D12_SHADER_BUFFER_DESC constantBufferDesc{};
        shaderReflectionConstantBuffer->GetDesc(&constantBufferDesc);

        const D3D12_ROOT_PARAMETER1 rootParameter
        {
            .ParameterType = D3D12_ROOT_PARAMETER_TYPE_CBV,
            .Descriptor{
                .ShaderRegister = shaderInputBindDesc.BindPoint,
                .RegisterSpace = shaderInputBindDesc.Space,
                .Flags = D3D12_ROOT_DESCRIPTOR_FLAG_NONE,
            },
        };
                
        rootParameters.push_back(rootParameter);
    }
}
```
The D3D12_SHADER_INPUT_BIND_DESC struct is defined as follows : 
```cpp
typedef struct _D3D12_SHADER_INPUT_BIND_DESC {
  LPCSTR                   Name;
  D3D_SHADER_INPUT_TYPE    Type;
  UINT                     BindPoint;
  UINT                     BindCount;
  UINT                     uFlags;
  D3D_RESOURCE_RETURN_TYPE ReturnType;
  D3D_SRV_DIMENSION        Dimension;
  UINT                     NumSamples;
  UINT                     Space;
  UINT                     uID;
} D3D12_SHADER_INPUT_BIND_DESC;
```
\
We check against the `Type` parameter to determine if said bound resources is a constant buffer, texture or any other resource type.
As binding resources require us to specify the root parameter index, I have a `std::unordered_map<std::wstring, int>` which simplifies this process by allowing me to set resources based on the name specified in the shader. With this map, all I need to do is give the name of the bound resource, and it gives back the root parameter index. In a practical scenario, this is how using it looks : 
```cpp
// The HLSL shader code.
struct SceneData
{
    row_major matrix viewProjectionMatrix;
};

struct TransformData
{
    row_major matrix modelMatrix;
};

ConstantBuffer<SceneData> sceneBuffer : register(b0);
ConstantBuffer<TransformData> transformBuffer : register(b1)
```

```cpp
// The C++ Application code (here, cmd is the ID3D12GraphicsCommandLists).
cmd->SetGraphicsRootConstantBufferView(graphicsPipeline->rootParameterIndexMap[L"sceneBuffer"],
    sceneBuffer.buffer->GetGPUVirtualAddress());

cmd->SetGraphicsRootConstantBufferView(graphicsPipeline->rootParameterIndexMap[L"transformBuffer"], transformBuffer. buffer->GetGPUVirtualAddress());
```
\
Reflecting textures is fairly similar to that of constant buffers.
> Note:exclamation:: Here I place each texture on its own descriptor table for simplicity. Please avoid this, and consider
coding up logic that will place multiple textures in a descriptor table.


```cpp
if (shaderInputBindDesc.Type == D3D_SIT_TEXTURE)
{
    // For now, each individual texture belongs in its own descriptor table. This can cause the root signature to quickly exceed the 64WORD size limit.
    rootParameterIndexMap[stringToWString(shaderInputBindDesc.Name)] = static_cast<uint32_t>(rootParameters.size());
    const CD3DX12_DESCRIPTOR_RANGE1 srvRange(D3D12_DESCRIPTOR_RANGE_TYPE_SRV,
                            1u,
                            shaderInputBindDesc.BindPoint,
                            shaderInputBindDesc.Space,
                            D3D12_DESCRIPTOR_RANGE_FLAG_DATA_STATIC);
            
    descriptorRanges.push_back(srvRange);

    const D3D12_ROOT_PARAMETER1 rootParameter
    {
        .ParameterType = D3D12_ROOT_PARAMETER_TYPE_DESCRIPTOR_TABLE,
        .DescriptorTable =
        {
            .NumDescriptorRanges = 1u,
            .pDescriptorRanges = &descriptorRanges.back(),
        },
        .ShaderVisibility = D3D12_SHADER_VISIBILITY_PIXEL,
    };

    rootParameters.push_back(rootParameter);
}
```

Once you are done with the reflection part, you can get the compiled shader bytecode into an IDxcBlob.
```cpp
Comptr<IDxcBlob> compiledShaderBlob{nullptr};
compiledShaderBuffer->GetOutput(DXC_OUT_OBJECT, IID_PPV_ARGS(&compiledShaderBlob), nullptr);

shader.shaderBlob = compiledShaderBlob;
```

With all this done, creating your root signature is heavily automated!
```cpp
const D3D12_VERSIONED_ROOT_SIGNATURE_DESC rootSignaureDesc = 
{
    .Version = D3D_ROOT_SIGNATURE_VERSION_1_1,
    .Desc_1_1 =
    {
        .NumParameters = static_cast<uint32_t>(rootParameters.size()),
        .pParameters = rootParameters.data(),
        .NumStaticSamplers = 0u,
        .pStaticSamplers = nullptr,
        .Flags = D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT,
    },
};
```

In most cases (i.e if you dont utilize static samplers), you could use the same reflection logic to automate the root signature creation process for *all* graphics and compute pipelines :metal:.

## Advantages of using Shader Reflection over Bindless Rendering
Unlike bindless rendering, where you need to check the descriptor heaps to view your data, shader reflection is used in collaboration with the traditional slot-based binding model. Due to this, debugging becomes very straightforward. In tools such as RenderDoc, you can directly view bound resources in the 'Pipeline State' section to see if anything is wrong.
![](/images/RenderDocVS.png)
![](/images/RenderDocPS.png)
Also, to use Bindless rendering, you need to set up the DirectXAgilitySDK and set the `D3D12_ROOT_SIGNATURE_FLAG_CBV_SRV_UAV_HEAP_DIRECTLY_INDEXED` and `D3D12_ROOT_SIGNATURE_FLAG_SAMPLER_HEAP_DIRECTLY_INDEXED` flags, which *may* not be available on all systems you are targetting.

## Should you use Bindless rendering instead?
> Note:exclamation:: If performance is critical, be sure to profile your code first before taking any major decisions.

While using shader reflection is *a lot* easier than manually modifying application/shader code, you might want to consider bindless rendering instead, as you can use a single root signature for all pipelines (both Graphics and Compute). With just a single constant buffer (that is a 32 bit root constant), modifying bound resources becomes *extremely* simple. 

In my personal opinion, have a look at Bindless Rendering (you can find my blog post on it [here](https://rtarun9.github.io/blogs/bindless_rendering/)) and see what works for you best. Shader reflection *may* be the better binding model on legacy hardware where indexing into the descriptor heap is inefficient, but bindless rendering can improve performance on some hardware. Again, PROFILE :red_circle:, PROFILE ::white_circle:, and PROFILE :large_blue_circle: if performance is critical!! :dizzy:

You might be interested in a 'Hybrid Binding Model' where you use Bindless rendering for your SRVs  / UAVs (i.e non CBV resources), and shader reflection for constant buffers, as some architectures have specific paths for them, and using bindless rendering for CBVs may not be compatible with them. Also, binding root Constant Buffer Views is fast in terms of CPU cost. You can find more details on this hybrid model [here](https://rtarun9.github.io/blogs/bindless_rendering/#performance-considerations).

## Closing Thoughts
While using modern low-level GPU APIs such as DX12 or Vulkan, I believe that using Bindless rendering provides way too many benefits over slot-based binding that is simplified by shader reflection.

However, On APIs such as DX11 where bindless rendering is not an option, shader reflection can be used to automate some of the manual stuff that can get in the way between you and some shiny pixels :sparkling_heart:
Vulkan users using GLSL can check out Khronos SPIRV-Reflect library, which is similar to using DXC for reflection. Do note that you can use HLSL with Vulkan by compiling HLSL to Spirv by setting the spirv flag in your compilation arguments. Doing so would require you to specify some additional vulkan attributes (such as `[[vk::binding(1, 1)]]`), but this means you can use the same shaders for both a DX12 and Vulkan backend ! (for the most part, unless you use features specific to one API).

Thank you so much for your time! Feel free to leave comments if you felt something was lacking/incorrect or your opinions on this simple blog post. If you would like to reach out to me, head over to the [Home page](/) to find some contact links there.

## More Detailed Resources
If you want to go deeper into shader reflection using DXC (and HLSL compilation using DXC), here are some resources I have found to be extremely helpful: \
[Simon Coenes's article on compiling HLSL shaders using DXC](https://simoncoenen.com/blog/programming/graphics/DxcCompiling) \
[Laptrinhx's blog on using the DXC C++ API](https://laptrinhx.com/using-the-directxshadercompiler-c-api-revised-1531596888/) \
[The official Microsoft DXC documentation](https://github.com/microsoft/DirectXShaderCompiler#documentation)
