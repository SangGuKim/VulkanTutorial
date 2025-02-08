# 멀티샘플링

## 소개

프로그램은 이제 텍스처에 대해 여러 세부 수준을 불러올 수 있으며, 이는 뷰어에서 멀리 떨어진 객체를 렌더링할 때 아티팩트를 수정합니다. 이미지는 이제 훨씬 부드러워졌지만, 자세히 살펴보면 그려진 기하학적 도형의 가장자리를 따라 톱니 모양의 패턴이 보입니다. 이는 우리가 초기 프로그램에서 쿼드를 렌더링했을 때 특히 눈에 띕니다:

![](/images/texcoord_visualization.png)

이 원치 않는 효과를 "앨리어싱"이라고 하며, 렌더링에 사용할 수 있는 픽셀 수가 제한되어 있기 때문에 발생합니다. 무한한 해상도를 가진 디스플레이는 존재하지 않으므로, 어느 정도는 항상 보일 것입니다. 이를 수정하는 몇 가지 방법이 있으며, 이 장에서는 가장 인기 있는 방법 중 하나인 [멀티샘플 안티앨리어싱](https://en.wikipedia.org/wiki/Multisample_anti-aliasing) (MSAA)에 초점을 맞출 것입니다.

일반적인 렌더링에서 픽셀 색상은 대부분의 경우 화면상의 대상 픽셀 중심에 있는 단일 샘플 지점을 기반으로 결정됩니다. 그려진 선의 일부가 특정 픽셀을 통과하지만 샘플 지점을 덮지 않으면 해당 픽셀은 비워져 "계단식" 효과를 초래합니다.

![](/images/aliasing.png)

MSAA가 하는 일은 픽셀당 여러 샘플 지점(이름에서 알 수 있듯이)을 사용하여 최종 색상을 결정하는 것입니다. 예상할 수 있듯이, 샘플이 많을수록 결과는 더 좋지만 계산 비용도 더 많이 듭니다.

![](/images/antialiasing.png)

우리의 구현에서는 사용 가능한 최대 샘플 수를 사용하는 데 중점을 둘 것입니다. 응용 프로그램에 따라 이것이 항상 최선의 접근 방법은 아닐 수 있으며, 최종 결과가 품질 요구 사항을 충족한다면 더 높은 성능을 위해 샘플 수를 줄이는 것이 더 낫습니다.

## 사용 가능한 샘플 수 가져오기

하드웨어에서 사용할 수 있는 샘플 수를 결정하기 위해 시작합시다. 대부분의 현대 GPU는 최소 8개의 샘플을 지원하지만, 이 숫자는 어디에서나 동일하다는 보장은 없습니다. 새로운 클래스 멤버를 추가하여 이를 추적할 것입니다:

```c++
...
VkSampleCountFlagBits msaaSamples = VK_SAMPLE_COUNT_1_BIT;
...
```

기본적으로 픽셀당 하나의 샘플만 사용할 것이며, 이는 멀티샘플링이 없는 것과 동일하므로 최종 이미지는 변경되지 않을 것입니다. 정확한 최대 샘플 수는 선택한 물리적 장치와 연관된 `VkPhysicalDeviceProperties`에서 추출할 수 있습니다. 우리는 깊이 버퍼를 사용하므로 색상과 깊이에 대한 샘플 수를 모두 고려해야 합니다. 둘 다 지원하는 가장 높은 샘플 수가 최대 지원 가능한 값이 될 것입니다. 이 정보를 가져오는 함수를 추가하세요:

```c++
VkSampleCountFlagBits getMaxUsableSampleCount() {
    VkPhysicalDeviceProperties physicalDeviceProperties;
    vkGetPhysicalDeviceProperties(physicalDevice, &physicalDeviceProperties);

    VkSampleCountFlags counts = physicalDeviceProperties.limits.framebufferColorSampleCounts & physicalDeviceProperties.limits.framebufferDepthSampleCounts;
    if (counts & VK_SAMPLE_COUNT_64_BIT) { return VK_SAMPLE_COUNT_64_BIT; }
    if (counts & VK_SAMPLE_COUNT_32_BIT) { return VK_SAMPLE_COUNT_32_BIT; }
    if (counts & VK_SAMPLE_COUNT_16_BIT) { return VK_SAMPLE_COUNT_16_BIT; }
    if (counts & VK_SAMPLE_COUNT_8_BIT) { return VK_SAMPLE_COUNT_8_BIT; }
    if (counts & VK_SAMPLE_COUNT_4_BIT) { return VK_SAMPLE_COUNT_4_BIT; }
    if (counts & VK_SAMPLE_COUNT_2_BIT) { return VK_SAMPLE_COUNT_2_BIT; }

    return VK_SAMPLE_COUNT_1_BIT;
}
```

이제 이 함수를 사용하여 물리적 장치 선택 과정에서 `msaaSamples` 변수를 설정할 것입니다. 이를 위해 `pickPhysicalDevice` 함수를 약간 수정해야 합니다:

```c++
void pickPhysicalDevice() {
    ...
    for (const auto& device : devices) {
        if (isDeviceSuitable(device)) {
            physicalDevice = device;
            msaaSamples = getMaxUsableSampleCount();
            break;
        }
    }
    ...
}
```

## 렌더 타겟 설정

MSAA에서는 각 픽셀이 화면에 렌더링되기 전에 오프스크린 버퍼에서 샘플링됩니다. 이 새 버퍼는 우리가 렌더링했던 일반 이미지와 약간 다릅니다. 픽셀당 하나 이상의 샘플을 저장할 수 있어야 합니다. 멀티샘플 버퍼가 생성되면 기본 프레임버퍼(픽셀당 하나의 샘플만 저장)로 해결해야 합니다. 이것이 우리가 추가 렌더 타겟을 생성하고 현재 드로잉 프로세스를 수정해야 하는 이유입니다. 깊이 버퍼와 마찬가지로 한 번에 하나의 드로잉 작업만 활성화되므로 하나의 렌더 타겟만 필요합니다. 다음 클래스 멤버를 추가하세요:

```c++
...
VkImage colorImage;
VkDeviceMemory colorImageMemory;
VkImageView colorImageView;
...
```

이 새 이미지는 픽셀당 원하는 샘플 수를 저장해야 하므로, 이미지 생성 과정에서 `VkImageCreateInfo`에 이 숫자를 전달해야 합니다. `createImage` 함수를 수정하여 `numSamples` 매개변수를 추가하세요:

```c++
void createImage(uint32_t width, uint32_t height, uint32_t mipLevels, VkSampleCountFlagBits numSamples, VkFormat format, VkImageTiling tiling, VkImageUsageFlags usage, VkMemoryPropertyFlags properties, VkImage& image, VkDeviceMemory& imageMemory) {
    ...
   

 imageInfo.samples = numSamples;
    ...
```

현재로서는 이 함수를 호출할 때 `VK_SAMPLE_COUNT_1_BIT`를 사용하여 모든 호출을 업데이트하세요 - 구현이 진행됨에 따라 적절한 값으로 이를 대체할 것입니다:

```c++
createImage(swapChainExtent.width, swapChainExtent.height, 1, VK_SAMPLE_COUNT_1_BIT, depthFormat, VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, depthImage, depthImageMemory);
...
createImage(texWidth, texHeight, mipLevels, VK_SAMPLE_COUNT_1_BIT, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_TRANSFER_SRC_BIT | VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, textureImage, textureImageMemory);
```

이제 멀티샘플 컬러 버퍼를 생성할 것입니다. `createColorResources` 함수를 추가하고 `createImage` 함수에 함수 매개변수로 `msaaSamples`를 사용하는 것을 주목하세요. 또한, 이미지가 픽셀당 하나 이상의 샘플을 가지는 경우 Vulkan 사양에서 강제하는 대로 한 개의 밉맵 레벨만 사용하고 있습니다. 또한, 이 컬러 버퍼는 텍스처로 사용되지 않을 것이므로 밉맵이 필요 없습니다:

```c++
void createColorResources() {
    VkFormat colorFormat = swapChainImageFormat;

    createImage(swapChainExtent.width, swapChainExtent.height, 1, msaaSamples, colorFormat, VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_TRANSIENT_ATTACHMENT_BIT | VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, colorImage, colorImageMemory);
    colorImageView = createImageView(colorImage, colorFormat, VK_IMAGE_ASPECT_COLOR_BIT, 1);
}
```

일관성을 유지하기 위해 `createDepthResources` 바로 전에 함수를 호출하세요:

```c++
void initVulkan() {
    ...
    createColorResources();
    createDepthResources();
    ...
}
```

이제 멀티샘플 컬러 버퍼가 준비되었으므로 깊이를 처리할 차례입니다. `createDepthResources`를 수정하고 깊이 버퍼에서 사용되는 샘플 수를 업데이트하세요:

```c++
void createDepthResources() {
    ...
    createImage(swapChainExtent.width, swapChainExtent.height, 1, msaaSamples, depthFormat, VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, depthImage, depthImageMemory);
    ...
}
```

이제 몇 가지 새로운 Vulkan 리소스를 생성했으므로 필요할 때 이들을 해제하는 것을 잊지 마세요:

```c++
void cleanupSwapChain() {
    vkDestroyImageView(device, colorImageView, nullptr);
    vkDestroyImage(device, colorImage, nullptr);
    vkFreeMemory(device, colorImageMemory, nullptr);
    ...
}
```

창 크기가 조절될 때 새 컬러 이미지가 올바른 해상도로 다시 생성될 수 있도록 `recreateSwapChain`을 업데이트하세요:

```c++
void recreateSwapChain() {
    ...
    createImageViews();
    createColorResources();
    createDepthResources();
    ...
}
```

초기 MSAA 설정을 완료했으므로 이제 이 새 리소스를 그래픽 파이프라인, 프레임버퍼, 렌더 패스에서 사용하고 결과를 확인해야 합니다!

## 새 어태치먼트 추가

먼저 렌더 패스를 처리하세요. `createRenderPass`를 수정하고 색상 및 깊이 어태치먼트 생성 정보 구조체를 업데이트하세요:

```c++
void createRenderPass() {
    ...
    colorAttachment.samples = msaaSamples;
    colorAttachment.finalLayout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
    ...
    depthAttachment.samples = msaaSamples;
    ...
```

`VK_IMAGE_LAYOUT_PRESENT_SRC_KHR에서 VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL로 finalLayout을 변경한 것을 알 수 있습니다. 멀티샘플 이미지는 직접 표시될 수 없기 때문입니다. 먼저 이를 일반 이미지로 변환해야 합니다. 이 요구사항은 깊이 버퍼에는 적용되지 않습니다, 왜냐하면 깊이 버퍼는 어떤 경우에도 표시될 일이 없기 때문입니다. 그러므로 이른바 해결 어태치먼트로 알려진 색상을 위한 새로운 어태치먼트 하나만 추가해야 합니다:

```c++
    ...
    VkAttachmentDescription colorAttachmentResolve{};
    colorAttachmentResolve.format = swapChainImageFormat;
    colorAttachmentResolve.samples = VK_SAMPLE_COUNT_1_BIT;
    colorAttachmentResolve.loadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    colorAttachmentResolve.storeOp = VK_ATTACHMENT_STORE_OP_STORE;
    colorAttachmentResolve.stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    colorAttachmentResolve.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
    colorAttachmentResolve.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
    colorAttachmentResolve.finalLayout = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR;
    ...
```

렌더 패스는 이제 멀티샘플 컬러 이미지를 일반 첨부 파일로 해결하도록 지시받아야 합니다. 새로운 첨부 참조를 생성하여 해결 대상으로 사용될 컬러 버퍼를 가리키게 합니다:

```c++
    ...
    VkAttachmentReference colorAttachmentResolveRef{};
    colorAttachmentResolveRef.attachment = 2;
    colorAttachmentResolveRef.layout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
    ...
```

`pResolveAttachments` 하위 패스 구조 멤버를 새로 생성된 첨부 참조를 가리키도록 설정하세요. 이것으로 렌더 패스는 멀티샘플 해결 작업을 정의할 수 있으며, 이를 통해 이미지를 화면에 렌더링할 수 있습니다:

```
    ...
    subpass.pResolveAttachments = &colorAttachmentResolveRef;
    ...
```

멀티샘플 컬러 이미지를 재사용하므로 `VkSubpassDependency`의 `srcAccessMask`를 업데이트하는 것이 필요합니다. 이 업데이트는 색상 첨부에 대한 모든 쓰기 작업이 완료된 후 후속 작업이 시작되도록 보장함으로써 쓰기 후 쓰기 위험을 방지하고 불안정한 렌더링 결과를 초래할 수 있습니다:

```c++
    ...
    dependency.srcAccessMask = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT | VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT;
    ...
```

이제 새 색상 첨부 파일로 렌더 패스 정보 구조를 업데이트하세요:

```c++
    ...
    std::array<VkAttachmentDescription, 3> attachments = {colorAttachment, depthAttachment, colorAttachmentResolve};
    ...
```

렌더 패스가 준비되었으므로 `createFramebuffers`를 수정하고 새 이미지 뷰를 목록에 추가하세요:

```c++
void createFramebuffers() {
        ...
        std::array<VkImageView, 3> attachments = {
            colorImageView,
            depthImageView,
            swapChainImageViews[i]
        };
        ...
}
```

새로 생성된 파이프라인이 둘 이상의 샘플을 사용하도록 설정하려면 `createGraphicsPipeline`을 수정하세요:

```c++
void createGraphicsPipeline() {
    ...
    multisampling.rasterizationSamples = msaaSamples;
    ...
}
```

이제 프로그램을 실행하면 다음과 같은 결과를 볼 수 있습니다:

![](/images/multisampling.png)

밉맵과 마찬가지로, 차이점이 바로 눈에 띄지 않을 수 있습니다. 자세히 보면 가장자리가 덜 톱니 모양이며 전체 이미지가

 원본에 비해 약간 부드러워 보입니다.

![](/images/multisampling_comparison.png)

차이점은 가장자리 중 하나를 가까이에서 볼 때 더 두드러집니다:

![](/images/multisampling_comparison2.png)

## 품질 개선

현재 MSAA 구현에는 더 자세한 장면에서 출력 이미지의 품질에 영향을 줄 수 있는 몇 가지 제한이 있습니다. 예를 들어, 현재는 셰이더 에일리어싱으로 인해 발생할 수 있는 잠재적 문제를 해결하지 않고 있습니다. 즉, MSAA는 기하학적 도형의 가장자리만 부드럽게 만들고 내부 채우기는 그대로 둡니다. 이로 인해 화면에 부드럽게 렌더링된 다각형이 있지만 대비가 높은 색상이 포함된 텍스처는 여전히 에일리어싱이 보일 수 있습니다. 이 문제를 해결하는 한 가지 방법은 [샘플 셰이딩](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap27.html#primsrast-sampleshading)을 활성화하는 것입니다. 이는 이미지 품질을 추가로 향상시킬 수 있지만 추가적인 성능 비용이 듭니다:

```c++

void createLogicalDevice() {
    ...
    deviceFeatures.sampleRateShading = VK_TRUE; // 디바이스에 샘플 셰이딩 기능을 활성화
    ...
}

void createGraphicsPipeline() {
    ...
    multisampling.sampleShadingEnable = VK_TRUE; // 파이프라인에서 샘플 셰이딩을 활성화
    multisampling.minSampleShading = .2f; // 샘플 셰이딩의 최소 비율; 1에 가까울수록 더 부드러움
    ...
}
```

이 예에서는 샘플 셰이딩을 비활성화하겠지만 특정 시나리오에서는 품질 개선이 눈에 띄게 나타날 수 있습니다:

![](/images/sample_shading.png)

## 결론

이 지점에 도달하기까지 많은 작업이 필요했지만, 이제 Vulkan 프로그램에 대한 좋은 기반을 갖추게 되었습니다. 이제 기본 Vulkan 원리에 대한 지식이 있으므로 더 많은 기능을 탐색하기 시작할 수 있습니다, 예를 들면:

* 푸시 상수
* 인스턴스 렌더링
* 동적 유니폼
* 별도의 이미지 및 샘플러 디스크립터
* 파이프라인 캐시
* 멀티스레드 명령 버퍼 생성
* 여러 서브패스
* 컴퓨트 셰이더

현재 프로그램은 많은 방법으로 확장될 수 있습니다, 예를 들어 Blinn-Phong 조명, 후처리 효과 및 그림자 매핑을 추가하는 것입니다. Vulkan의 명시성에도 불구하고 많은 개념이 여전히 동일하게 작동하기 때문에 다른 API의 튜토리얼에서 이러한 효과가 어떻게 작동하는지 배울 수 있어야 합니다.

[C++ 코드](/code/30_multisampling.cpp) /
[버텍스 셰이더](/code/27_shader_depth.vert) /
[프래그먼트 셰이더](/code/27_shader_depth.frag)