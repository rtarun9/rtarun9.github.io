---
title: "Helios [C++20 DirectX12 renderer]"
draft: false
weight: 1
cover:
    image: "/images/IBL1.png"
description: "DirectX12 Graphics renderer featuring rendering techniques such as PBR / IBL, Shadow Mapping, Deferred Shading, etc."
summary: "DirectX12 Graphics renderer featuring rendering techniques such as PBR / IBL, Shadow Mapping, Deferred Shading, etc."
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
* Screen Space Ambient Occlusion (SSAO)
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

> Screen Space Ambient Occlusion
![](/images/SSAO.png)

> Editor (using ImGui)
![](/images/Editor1.png)

> Deferred Shading
![](/images/Deferred1.png)

## Example code using the DX12 abstractions:
```cpp
// The code loads assets in a multithreaded fashion and applied PBR, IBL, SSAO, DeferredShading and PCF Ominidirectional shadow mapping.
#include "Helios.hpp"

using namespace helios;

class SandBox final : public helios::core::Application
{
  public:
    explicit SandBox(const std::string_view windowTitle) : Application(windowTitle)
    {
    }

    void loadContent() override
    {
        loadScene();

        loadTextures();

        loadPipelineStates();

        m_postProcessingBuffer = m_graphicsDevice->createBuffer<interlop::PostProcessingBuffer>(gfx::BufferCreationDesc{
            .usage = gfx::BufferUsage::ConstantBuffer,
            .name = L"Post Processing Buffer",
        });

        m_deferredGPass =
            std::make_unique<rendering::DeferredGeometryPass>(m_graphicsDevice.get(), m_windowWidth, m_windowHeight);

        m_ibl = std::make_unique<rendering::IBL>(m_graphicsDevice.get());

        m_irradianceTexture =
            m_ibl->generateIrradianceTexture(m_graphicsDevice.get(), m_scene->m_cubeMap->m_cubeMapTexture);

        m_prefilterTexture =
            m_ibl->generatePrefilterTexture(m_graphicsDevice.get(), m_scene->m_cubeMap->m_cubeMapTexture);

        m_brdfLUTTexture = m_ibl->generateBRDFLutTexture(m_graphicsDevice.get());

        m_shadowMappingPass = std::make_unique<rendering::PCFShadowMappingPass>(m_graphicsDevice.get());

        m_ssaoPass = std::make_unique<rendering::SSAOPass>(m_graphicsDevice.get(), m_windowWidth, m_windowHeight);
    }

    void loadScene()
    {
        m_scene->addModel(m_graphicsDevice.get(),
                          scene::ModelCreationDesc{
                              .modelPath = L"Assets/Models/DamagedHelmet/glTF/DamagedHelmet.gltf",
                              .modelName = L"Damaged Helmet",
                          });

        m_scene->addModel(m_graphicsDevice.get(),
                          scene::ModelCreationDesc{
                              .modelPath = L"Assets/Models/MetalRoughSpheres/glTF/MetalRoughSpheres.gltf",
                              .modelName = L"MetalRough spheres",
                          });

        m_scene->addModel(m_graphicsDevice.get(), scene::ModelCreationDesc{
                                                      .modelPath = L"Assets/Models/Sponza/sponza.glb",
                                                      .modelName = L"Sponza",
                                                      .scale =
                                                          {
                                                              0.1f,
                                                              0.1f,
                                                              0.1f,
                                                          },
                                                  });
  
        m_scene->addLight(m_graphicsDevice.get(),
                          scene::LightCreationDesc{.lightType = scene::LightTypes::DirectionalLightData});

        m_scene->addLight(m_graphicsDevice.get(),
                          scene::LightCreationDesc{.lightType = scene::LightTypes::PointLightData});

        m_scene->addCubeMap(m_graphicsDevice.get(),
                            scene::CubeMapCreationDesc{
                                .equirectangularTexturePath = L"Assets/Textures/Environment.hdr",
                                .name = L"Environment Cube Map",
                            });
    }

    void loadPipelineStates()
    {

        m_pipelineState = m_graphicsDevice->createPipelineState(gfx::GraphicsPipelineStateCreationDesc{
            .shaderModule =
                {
                    .vertexShaderPath = L"Shaders/Shading/PBR.hlsl",
                    .pixelShaderPath = L"Shaders/Shading/PBR.hlsl",
                },
            .depthFormat = DXGI_FORMAT_UNKNOWN,
            .pipelineName = L"PBR Pipeline",
        });

        m_postProcessingPipelineState = m_graphicsDevice->createPipelineState(gfx::GraphicsPipelineStateCreationDesc{
            .shaderModule =
                {
                    .vertexShaderPath = L"Shaders/PostProcessing/PostProcessing.hlsl",
                    .pixelShaderPath = L"Shaders/PostProcessing/PostProcessing.hlsl",
                },
            .rtvFormats = {DXGI_FORMAT_R10G10B10A2_UNORM},
            .rtvCount = 1u,
            .depthFormat = DXGI_FORMAT_D32_FLOAT,
            .pipelineName = L"Post Processing Pipeline",
        });

        m_fullScreenTrianglePassPipelineState =
            m_graphicsDevice->createPipelineState(gfx::GraphicsPipelineStateCreationDesc{
                .shaderModule =
                    {
                        .vertexShaderPath = L"Shaders/RenderPass/FullScreenTrianglePass.hlsl",
                        .pixelShaderPath = L"Shaders/RenderPass/FullScreenTrianglePass.hlsl",
                    },
                .rtvFormats = {DXGI_FORMAT_R10G10B10A2_UNORM},
                .rtvCount = 1u,
                .depthFormat = DXGI_FORMAT_UNKNOWN,
                .pipelineName = L"Full Screen Triangle Pass Pipeline",
            });
    }

    void loadTextures()
    {
        static constexpr std::array<uint32_t, 3u> indices = {
            0u,
            1u,
            2u,
        };

        m_renderTargetIndexBuffer = m_graphicsDevice->createBuffer<uint32_t>(
            gfx::BufferCreationDesc{
                .usage = gfx::BufferUsage::IndexBuffer,
                .name = L"Render Target Index Buffer",
            },
            indices);

        m_depthTexture = m_graphicsDevice->createTexture(gfx::TextureCreationDesc{
            .usage = gfx::TextureUsage::DepthStencil,
            .width = m_windowWidth,
            .height = m_windowHeight,
            .format = DXGI_FORMAT_D32_FLOAT,
            .name = L"Depth Texture",
        });

        m_fullScreenPassDepthTexture = m_graphicsDevice->createTexture(gfx::TextureCreationDesc{
            .usage = gfx::TextureUsage::DepthStencil,
            .width = m_windowWidth,
            .height = m_windowHeight,
            .format = DXGI_FORMAT_D32_FLOAT,
            .name = L"Full Screen Pass Depth Texture",
        });

        m_offscreenRenderTarget = m_graphicsDevice->createTexture(gfx::TextureCreationDesc{
            .usage = gfx::TextureUsage::RenderTarget,
            .width = m_windowWidth,
            .height = m_windowHeight,
            .format = DXGI_FORMAT_R16G16B16A16_FLOAT,
            .name = L"OffScreen Render Target",
        });

        m_postProcessingRenderTarget = m_graphicsDevice->createTexture(gfx::TextureCreationDesc{
            .usage = gfx::TextureUsage::RenderTarget,
            .width = m_windowWidth,
            .height = m_windowHeight,
            .format = DXGI_FORMAT_R10G10B10A2_UNORM,
            .name = L"Post Processing Render Target",
        });
    }

    void update(const float deltaTime) override
    {
        m_scene->update(deltaTime, m_input, static_cast<float>(m_windowWidth) / m_windowHeight);
    }

    void render() override
    {
        m_graphicsDevice->beginFrame();

        std::unique_ptr<gfx::GraphicsContext>& gctx = m_graphicsDevice->getCurrentGraphicsContext();
        gfx::Texture& currentBackBuffer = m_graphicsDevice->getCurrentBackBuffer();

        // const std::array<float, 4> clearColor = {std::abs(std::cosf(m_frameCount / 120.0f)), 0.0f,
        //                                          std::abs(std::sinf(m_frameCount / 120.0f)), 1.0f};
        static std::array<float, 4> clearColor = {0.0f, 0.0f, 0.0f, 1.0f};

        gctx->clearRenderTargetView(m_offscreenRenderTarget, clearColor);
        gctx->clearRenderTargetView(m_postProcessingRenderTarget, clearColor);

        gctx->clearDepthStencilView(m_depthTexture);
        gctx->clearDepthStencilView(m_fullScreenPassDepthTexture);

        // RenderPass 0 : Deferred GPass.
        {
            m_deferredGPass->render(m_scene.get(), gctx.get(), m_depthTexture, m_windowWidth, m_windowHeight);
        }

        // RenderPass 2 : SSAO Pass.
        {
            interlop::SSAORenderResources renderResources = {
                .positionTextureIndex = m_deferredGPass->m_gBuffer.positionEmissiveRT.srvIndex,
                .normalTextureIndex = m_deferredGPass->m_gBuffer.normalEmissiveRT.srvIndex,
                .sceneBufferIndex = m_scene->m_sceneBuffer.cbvIndex,
            };

            m_ssaoPass->render(gctx.get(), m_renderTargetIndexBuffer, renderResources, m_windowWidth, m_windowHeight);
        }

        // RenderPass 3 : Shadow mapping pass.
        {
            m_shadowMappingPass->render(m_scene.get(), gctx.get());
        }

        // RenderPass 4 : Lighting Pass
        {
            gctx->setGraphicsRootSignatureAndPipeline(m_pipelineState);
            gctx->setRenderTarget(m_offscreenRenderTarget, m_fullScreenPassDepthTexture);
            gctx->setViewport(D3D12_VIEWPORT{
                .TopLeftX = 0.0f,
                .TopLeftY = 0.0f,
                .Width = static_cast<float>(m_windowWidth),
                .Height = static_cast<float>(m_windowHeight),
                .MinDepth = 0.0f,
                .MaxDepth = 1.0f,
            });

            gctx->setPrimitiveTopologyLayout(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);

            interlop::PBRRenderResources renderResources = {
                .albedoGBufferIndex = m_deferredGPass->m_gBuffer.albedoRT.srvIndex,
                .positionEmissiveGBufferIndex = m_deferredGPass->m_gBuffer.positionEmissiveRT.srvIndex,
                .normalEmissiveGBufferIndex = m_deferredGPass->m_gBuffer.normalEmissiveRT.srvIndex,
                .aoMetalRoughnessEmissiveGBufferIndex = m_deferredGPass->m_gBuffer.aoMetalRoughnessEmissiveRT.srvIndex,
                .irradianceTextureIndex = m_irradianceTexture.srvIndex,
                .prefilterTextureIndex = m_prefilterTexture.srvIndex,
                .brdfLUTTextureIndex = m_brdfLUTTexture.srvIndex,
                .shadowBufferIndex = m_shadowMappingPass->m_shadowBuffer.cbvIndex,
                .shadowDepthTextureIndex = m_shadowMappingPass->m_shadowDepthBuffer.srvIndex,
                .blurredSSAOTextureIndex = m_ssaoPass->m_blurSSAOTexture.srvIndex,
            };

            m_scene->renderModels(gctx.get(), renderResources);
        }

        // RenderPass 5 : Render lights and cube map using forward rendering.
        {
            gctx->setViewport(D3D12_VIEWPORT{
                .TopLeftX = 0.0f,
                .TopLeftY = 0.0f,
                .Width = static_cast<float>(m_windowWidth),
                .Height = static_cast<float>(m_windowHeight),
                .MinDepth = 0.0f,
                .MaxDepth = 1.0f,
            });

            gctx->setPrimitiveTopologyLayout(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
            gctx->setRenderTarget(m_offscreenRenderTarget, m_depthTexture);

            m_scene->renderLights(gctx.get());

            m_scene->renderCubeMap(gctx.get());
        }

        // RenderPass 6 : Post Processing Stage:
        {
            gctx->addResourceBarrier(m_offscreenRenderTarget.allocation.resource.Get(),
                                     D3D12_RESOURCE_STATE_RENDER_TARGET, D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE);
            gctx->executeResourceBarriers();

            gctx->setGraphicsRootSignatureAndPipeline(m_postProcessingPipelineState);
            gctx->setViewport(D3D12_VIEWPORT{
                .TopLeftX = 0.0f,
                .TopLeftY = 0.0f,
                .Width = static_cast<float>(m_windowWidth),
                .Height = static_cast<float>(m_windowHeight),
                .MinDepth = 0.0f,
                .MaxDepth = 1.0f,
            });

            gctx->setPrimitiveTopologyLayout(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
            gctx->setRenderTarget(m_postProcessingRenderTarget, m_fullScreenPassDepthTexture);

            interlop::PostProcessingRenderResources renderResources = {
                .renderTextureIndex = m_offscreenRenderTarget.srvIndex,
            };

            gctx->set32BitGraphicsConstants(&renderResources);
            gctx->setIndexBuffer(m_renderTargetIndexBuffer);
            gctx->drawInstanceIndexed(3u);
        }

        // Render pass 7 : Render post processing render target to swapchain backbuffer via a full screen triangle pass.
        {
            gctx->addResourceBarrier(m_offscreenRenderTarget.allocation.resource.Get(),
                                     D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE, D3D12_RESOURCE_STATE_RENDER_TARGET);

            gctx->addResourceBarrier(m_postProcessingRenderTarget.allocation.resource.Get(),
                                     D3D12_RESOURCE_STATE_RENDER_TARGET, D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE);

            gctx->addResourceBarrier(currentBackBuffer.allocation.resource.Get(), D3D12_RESOURCE_STATE_PRESENT,
                                     D3D12_RESOURCE_STATE_RENDER_TARGET);

            gctx->executeResourceBarriers();

            gctx->clearRenderTargetView(currentBackBuffer, clearColor);

            gctx->setGraphicsRootSignatureAndPipeline(m_fullScreenTrianglePassPipelineState);
            gctx->setViewport(D3D12_VIEWPORT{
                .TopLeftX = 0.0f,
                .TopLeftY = 0.0f,
                .Width = static_cast<float>(m_windowWidth),
                .Height = static_cast<float>(m_windowHeight),
                .MinDepth = 0.0f,
                .MaxDepth = 1.0f,
            });

            gctx->setPrimitiveTopologyLayout(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
            gctx->setRenderTarget(currentBackBuffer);

            interlop::FullScreenTrianglePassRenderResources renderResources = {
                .renderTextureIndex = m_postProcessingRenderTarget.srvIndex,
            };

            gctx->set32BitGraphicsConstants(&renderResources);
            gctx->setIndexBuffer(m_renderTargetIndexBuffer);
            gctx->drawInstanceIndexed(3u);

            m_editor->render(m_graphicsDevice.get(), m_scene.get(), m_deferredGPass->m_gBuffer,
                             m_shadowMappingPass.get(), m_ssaoPass.get(), m_postProcessingBufferData,
                             m_postProcessingRenderTarget, gctx.get());

            gctx->addResourceBarrier(m_postProcessingRenderTarget.allocation.resource.Get(),
                                     D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE, D3D12_RESOURCE_STATE_RENDER_TARGET);

            gctx->addResourceBarrier(currentBackBuffer.allocation.resource.Get(), D3D12_RESOURCE_STATE_RENDER_TARGET,
                                     D3D12_RESOURCE_STATE_PRESENT);

            gctx->executeResourceBarriers();
        }

        const std::array<gfx::Context* const, 1u> contexts = {
            gctx.get(),
        };

        m_graphicsDevice->getDirectCommandQueue()->executeContext(contexts);

        m_graphicsDevice->present();
        m_graphicsDevice->endFrame();

        m_frameCount++;
    }

  private:
    gfx::Texture m_offscreenRenderTarget{};
    gfx::Texture m_postProcessingRenderTarget{};

    gfx::PipelineState m_pipelineState{};
    gfx::PipelineState m_postProcessingPipelineState{};
    gfx::PipelineState m_fullScreenTrianglePassPipelineState{};

    gfx::Texture m_depthTexture{};
    gfx::Texture m_fullScreenPassDepthTexture{};

    gfx::Buffer m_renderTargetIndexBuffer{};

    gfx::Buffer m_postProcessingBuffer{};
    interlop::PostProcessingBuffer m_postProcessingBufferData{};

    std::unique_ptr<rendering::DeferredGeometryPass> m_deferredGPass{};
    std::unique_ptr<rendering::IBL> m_ibl{};
    std::unique_ptr<rendering::PCFShadowMappingPass> m_shadowMappingPass{};
    std::unique_ptr<rendering::SSAOPass> m_ssaoPass{};

    gfx::Texture m_irradianceTexture{};
    gfx::Texture m_prefilterTexture{};
    gfx::Texture m_brdfLUTTexture{};

    uint64_t m_frameCount{};
};

int main()
{
    SandBox sandbox{"Helios::SandBox"};
    sandbox.run();

    return 0;
}
```