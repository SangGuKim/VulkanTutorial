# 깊이 버퍼링

## 소개

지금까지 작업한 기하학적 구조는 3D로 투영되지만 완전히 평평합니다. 이 장에서는 위치에 Z 좌표를 추가하여 3D 메시를 준비할 것입니다. 이 세 번째 좌표를 사용하여 현재의 정사각형 위에 다른 정사각형을 배치하여 깊이별로 정렬되지 않은 기하학적 구조가 있을 때 발생하는 문제를 확인할 것입니다.

## 3D 기하학

`Vertex` 구조체를 3D 벡터를 사용하는 위치로 변경하고, 해당 `VkVertexInputAttributeDescription`의 `format`을 업데이트하세요:

```c++
struct Vertex {
    glm::vec3 pos;
    glm::vec3 color;
    glm::vec2 texCoord;

    ...

    static std::array<VkVertexInputAttributeDescription, 3> getAttributeDescriptions() {
        std::array<VkVertexInputAttributeDescription, 3> attributeDescriptions{};

        attributeDescriptions[0].binding = 0;
        attributeDescriptions[0].location = 0;
        attributeDescriptions[0].format = VK_FORMAT_R32G32B32_SFLOAT;
        attributeDescriptions[0].offset = offsetof(Vertex, pos);

        ...
    }
};
```

다음으로, 버텍스 셰이더를 업데이트하여 3D 좌표를 입력으로 받아 변환하도록 합니다. 이후에 재컴파일하는 것을 잊지 마세요!

```glsl
layout(location = 0) in vec3 inPosition;

...

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 1.0);
    fragColor = inColor;
    fragTexCoord = inTexCoord;
}
```

마지막으로, `vertices` 컨테이너를 업데이트하여 Z 좌표를 포함시키세요:

```c++
const std::vector<Vertex> vertices = {
    {{-0.5f, -0.5f, 0.0f}, {1.0f, 0.0f, 0.0f}, {0.0f, 0.0f}},
    {{0.5f, -0.5f, 0.0f}, {0.0f, 1.0f, 0.0f}, {1.0f, 0.0f}},
    {{0.5f, 0.5f, 0.0f}, {0.0f, 0.0f, 1.0f}, {1.0f, 1.0f}},
    {{-0.5f, 0.5f, 0.0f}, {1.0f, 1.0f, 1.0f}, {0.0f, 1.0f}}
};
```

이제 애플리케이션을 실행하면 이전과 동일한 결과를 볼 수 있습니다. 이제 장면을 더 흥미롭게 만들기 위해 추가적인 기하학을 추가하고, 이 장에서 해결하려는 문제를 보여줄 시간입니다. 현재 정사각형 바로 아래에 위치할 정사각형의 위치를 정의하기 위해 정점을 중복하세요:

![](/images/extra_square.svg)

Z 좌표를 `-0.5f`로 사용하고 추가 정사각형을 위한 적절한 인덱스를 추가하세요:

```c++
const std::vector<Vertex> vertices = {
    {{-0.5f, -0.5f, 0.0f}, {1.0f, 0.0f, 0.0f}, {0.0f, 0.0f}},
    {{0.5f, -0.5f, 0.0f}, {0.0f, 1.0f, 0.0f}, {1.0f, 0.0f}},
    {{0.5f, 0.5f, 0.0f}, {0.0f, 0.0f, 1.0f}, {1.0f, 1.0f}},
    {{-0.5f, 0.5f, 0.0f}, {1.0f, 1.0f, 1.0f}, {0.0f, 1.0f}},

    {{-0.5f, -0.5f, -0.5f}, {1.0f, 0.0f, 0.0f}, {0.0f, 0.0f}},
    {{0.5f, -0.5f, -0.5f}, {0.0f, 1.0f, 0.0f}, {1.0f, 0.0f}},
    {{0.5f, 0.5f, -0.5f}, {0.0f, 0.0f, 1.0f}, {1.0f, 1.0f}},
    {{-0.5f, 0.5f, -0.5f}, {1.0f, 1.0f, 1.0f}, {0.0f, 1.0f}}
};

const std::vector<uint16_t> indices = {
    0, 1, 2, 2, 3, 0,
    4, 5, 6, 6, 7, 4
};
```

이제 프로그램을 실행하면 Escher의 일러스트레이션을 연상시키는 무언가를 볼 수 있습니다:

![](/images/depth_issues.png)

문제는 하단 정사각형의 프래그먼트가 인덱스 배열에서 뒤에 나오기 때문에 상단 정사각형의 프래그먼트 위에 그려진다는 것입니다. 이 문제를 해결하는 두 가지 방법이 있습니다:

* 모든 드로우 호출을 뒤에서 앞으로 깊이별로 정렬
* 깊이 버퍼를 사용한 깊이 테스트

첫 번째 방법은 투명 객체를 그릴 때 일반적으로 사용되며, 순서 독립적 투명도는 해결하기 어려운 도전입니다. 그러나 깊이별로 프래그먼트를 정렬하는 문제는 *깊이 버퍼*를 사용하여 훨씬 더 일반적으로 해결됩니다. 깊이 버퍼는 모든 위치의 깊이를 저장하는 추가적인 첨부 파일로, 색상 첨부 파일이 모든 위치의 색상을 저장하는 것과 같습니다. 래스터라이저가 프래그먼트를 생성할 때마다, 깊이 테스트는 새 프래그먼트가 이전 것보다 가까운지를 확인합니다. 그렇지 않다면, 새 프래그먼트는 버려집니다. 깊이 테스트를 통과한 프래그먼트는 자신의 깊이를 깊이 버퍼에 기록합니다. 프래그먼트 셰이더에서 색상 출력을 조작할 수 있는 것처럼 이 값을 조작할 수 있습니다.

```c++
#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEPTH_ZERO_TO_ONE
#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>
```

GLM에 의해 생성된 관점 투영 행렬은 기본적으로 OpenGL의 깊이 범위 `-1.0`에서 `1.0`을 사용합니다. 우리는 Vulkan 범위 `0.0`에서 `1.0`을 사용하도록 `GLM_FORCE_DEPTH_ZERO_TO_ONE` 정의를 사용하여 구성해야 합니다.

## 깊이 이미지 및 뷰

깊이 첨부 파일은 색상

 첨부 파일과 마찬가지로 이미지를 기반으로 합니다. 차이점은 스왑 체인이 자동으로 깊이 이미지를 생성하지 않는다는 것입니다. 동시에 실행되는 드로우 작업은 하나뿐이므로 하나의 깊이 이미지만 필요합니다. 깊이 이미지는 다시 이미지, 메모리 및 이미지 뷰의 삼박자를 필요로 합니다.

```c++
VkImage depthImage;
VkDeviceMemory depthImageMemory;
VkImageView depthImageView;
```

이러한 리소스를 설정하기 위해 새로운 함수 `createDepthResources`를 생성하세요:

```c++
void initVulkan() {
    ...
    createCommandPool();
    createDepthResources();
    createTextureImage();
    ...
}

...

void createDepthResources() {

}
```

깊이 이미지를 생성하는 것은 비교적 간단합니다. 스왑 체인 범위에 의해 정의된 색상 첨부 파일과 동일한 해상도를 가져야 하며, 깊이 첨부 파일에 적합한 이미지 사용, 최적 타일링 및 디바이스 로컬 메모리를 가져야 합니다. 유일한 질문은 깊이 이미지에 적합한 형식은 무엇인가 하는 것입니다. 형식은 `_D??_`로 표시된 깊이 구성 요소를 포함해야 합니다.

텍스처 이미지와 달리 프로그램에서 텍셀을 직접 액세스할 필요가 없기 때문에 특정 형식이 필요하지 않습니다. 단지 실제 애플리케이션에서 흔히 사용되는 최소 24비트의 합리적인 정확도를 가져야 합니다. 이 요구 사항에 맞는 여러 형식이 있습니다:

* `VK_FORMAT_D32_SFLOAT`: 깊이를 위한 32비트 float
* `VK_FORMAT_D32_SFLOAT_S8_UINT`: 깊이를 위한 32비트 부호 있는 float 및 8비트 스텐실 구성 요소
* `VK_FORMAT_D24_UNORM_S8_UINT`: 깊이를 위한 24비트 float 및 8비트 스텐실 구성 요소

스텐실 구성 요소는 [스텐실 테스트](https://en.wikipedia.org/wiki/Stencil_buffer)에 사용되며, 이는 깊이 테스트와 결합할 수 있는 추가적인 테스트입니다. 이에 대해서는 향후 장에서 살펴볼 것입니다.

우리는 매우 흔하게 지원되는 `VK_FORMAT_D32_SFLOAT` 형식을 간단히 사용할 수 있습니다(하드웨어 데이터베이스 참조), 하지만 가능한 경우 애플리케이션에 추가적인 유연성을 추가하는 것이 좋습니다. 우리는 가장 바람직한 것부터 가장 적게 바람직한 순서로 후보 형식 목록을 취하는 `findSupportedFormat` 함수를 작성할 것입니다. 지원되는 첫 번째 형식을 확인합니다:

```c++
VkFormat findSupportedFormat(const std::vector<VkFormat>& candidates, VkImageTiling tiling, VkFormatFeatureFlags features) {

}
```

형식의 지원 여부는 타일링 모드와 사용에 따라 달라지므로 이러한 매개변수도 포함해야 합니다. 형식의 지원 여부는 `vkGetPhysicalDeviceFormatProperties` 함수를 사용하여 조회할 수 있습니다:

```c++
for (VkFormat format : candidates) {
    VkFormatProperties props;
    vkGetPhysicalDeviceFormatProperties(physicalDevice, format, &props);
}
```

`VkFormatProperties` 구조체에는 세 개의 필드가 있습니다:

* `linearTilingFeatures`: 선형 타일링으로 지원되는 사용 사례
* `optimalTilingFeatures`: 최적 타일링으로 지원되는 사용 사례
* `bufferFeatures`: 버퍼에 대해 지원되는 사용 사례

여기서는 첫 두 가지만 관련이 있으며, 우리가 확인하는 것은 함수의 `tiling` 매개변수에 따라 달라집니다:

```c++
if (tiling == VK_IMAGE_TILING_LINEAR && (props.linearTilingFeatures & features) == features) {
    return format;
} else if (tiling == VK_IMAGE_TILING_OPTIMAL && (props.optimalTilingFeatures & features) == features) {
    return format;
}
```

원하는 사용에 대한 지원이 없는 경우 후보 형식 중 하나도 없다면 특별한 값을 반환하거나 간단히 예외를 발생시킬 수 있습니다:

```c++
VkFormat findSupportedFormat(const std::vector<VkFormat>& candidates, VkImageTiling tiling, VkFormatFeatureFlags features) {
    for (VkFormat format : candidates) {
        VkFormatProperties props;
        vkGetPhysicalDeviceFormatProperties(physicalDevice, format, & props);

        if (tiling == VK_IMAGE_TILING_LINEAR && (props.linearTilingFeatures & features) == features) {
            return format;
        } else if (tiling == VK_IMAGE_TILING_OPTIMAL && (props.optimalTilingFeatures & features) == features) {
            return format;
        }
    }

    throw std::runtime_error("failed to find supported format!");
}
```

이제 깊이 첨부 파일로 사용할 깊이 구성 요소가 지원되는 형식을 선택하기 위해 `findDepthFormat` 도우미 함수를 사용하여 이 함수를 호출할 것입니다:

```c++
VkFormat findDepthFormat() {
    return findSupportedFormat(
        {VK_FORMAT_D32_SFLOAT, VK_FORMAT_D32_SFLOAT_S8_UINT, VK_FORMAT_D24_UNORM_S8_UINT},
        VK_IMAGE_TILING_OPTIMAL,
        VK_FORMAT_FEATURE_DEPTH_STENCIL_ATTACHMENT_BIT
    );
}
```

이 경우 `VK_IMAGE_USAGE_` 대신 `VK_FORMAT_FEATURE_` 플래그를 사용해야 합니다. 이 모든 후보 형식은 깊이 구성 요소를 포함하지만, 후자 두 가지는 또한 스텐실 구성 요소를 포함합니다. 아직 그것을 사용하지는 않지만, 이러한 형식의 이미지에 레이아웃 전환을 수행할 때 그것을 고려해야 합니다. 선택한 깊이 형식이 스텐실 구성 요소를 포함하는지 알려주는 간단한 도우미 함수를 추가하세요:

```c++
bool hasStencilComponent(VkFormat format) {
    return format == VK_FORMAT_D32_SFLOAT_S8_UINT || format == VK_FORMAT_D24_UNORM_S8_UINT;
}
```

`createDepthResources`에서 깊이 형식을 찾는 함수를 호출하세요:

```c++
VkFormat depthFormat = findDepthFormat();
```

이제 `createImage` 및 `createImageView` 도우미 함수를 호출하는 데 필요한 모든 정보를 갖추었습니다:

```c++
createImage(swapChainExtent.width, swapChainExtent.height, depthFormat, VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, depthImage, depthImageMemory);
depthImageView = createImageView(depthImage, depthFormat);
```

그러나 `createImageView` 함수는 현재 하위 리소스가 항상 `VK_IMAGE_ASPECT_COLOR_BIT`라고 가정하므로 해당 필드를 매개변수로 전환해야 합니다:

```c++
VkImageView createImageView(VkImage image, VkFormat format, VkImageAspectFlags aspectFlags) {
    ...
    viewInfo.subresourceRange.aspectMask = aspectFlags;
    ...
}
```

이 함수를 호출할 때 올바른 양상을 사용하여 모든 호출을 업데이트하세요:

```c++
swapChainImageViews[i] = createImageView(swapChainImages[i], swapChainImageFormat, VK_IMAGE_ASPECT_COLOR_BIT);
...
depthImageView = createImageView(depthImage,

 depthFormat, VK_IMAGE_ASPECT_DEPTH_BIT);
...
textureImageView = createImageView(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_ASPECT_COLOR_BIT);
```

깊이 이미지 생성이 완료되었습니다. 색상 첨부 파일처럼 렌더 패스 시작 시 클리어할 것이므로, 다른 이미지로 매핑하거나 복사할 필요는 없습니다.

### 깊이 이미지의 명시적 전환

렌더 패스에서 이 작업을 처리할 것이므로 이미지의 레이아웃을 깊이 첨부 파일로 명시적으로 전환할 필요는 없습니다. 그러나 완전성을 위해 이 섹션에서는 그 과정을 여전히 설명할 것입니다. 원한다면 건너뛸 수 있습니다.

`createDepthResources` 함수의 끝에서 `transitionImageLayout`을 호출하세요:

```c++
transitionImageLayout(depthImage, depthFormat, VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL);
```

기존 깊이 이미지 내용이 중요하지 않기 때문에 초기 레이아웃으로 정의되지 않은 레이아웃을 사용할 수 있습니다. `transitionImageLayout`의 일부 로직을 올바른 하위 리소스 양상을 사용하도록 업데이트해야 합니다:

```c++
if (newLayout == VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL) {
    barrier.subresourceRange.aspectMask = VK_IMAGE_ASPECT_DEPTH_BIT;

    if (hasStencilComponent(format)) {
        barrier.subresourceRange.aspectMask |= VK_IMAGE_ASPECT_STENCIL_BIT;
    }
} else {
    barrier.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
}
```

우리는 스텐실 구성 요소를 사용하지 않지만, 깊이 이미지의 레이아웃 전환에는 포함해야 합니다.

마지막으로 올바른 액세스 마스크와 파이프라인 단계를 추가하세요:

```c++
if (oldLayout == VK_IMAGE_LAYOUT_UNDEFINED && newLayout == VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL) {
    barrier.srcAccessMask = 0;
    barrier.dstAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;

    sourceStage = VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT;
    destinationStage = VK_PIPELINE_STAGE_TRANSFER_BIT;
} else if (oldLayout == VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL and newLayout == VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL) {
    barrier.srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
    barrier.dstAccessMask = VK_ACCESS_SHADER_READ_BIT;

    sourceStage = VK_PIPELINE_STAGE_TRANSFER_BIT;
    destinationStage = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
} else if (oldLayout == VK_IMAGE_LAYOUT_UNDEFINED and newLayout == VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL) {
    barrier.srcAccessMask = 0;
    barrier.dstAccessMask = VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_READ_BIT | VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT;

    sourceStage = VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT;
    destinationStage = VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT;
} else {
    throw std::invalid_argument("unsupported layout transition!");
}
```

깊이 버퍼는 프래그먼트가 보이는지 확인하기 위해 깊이 테스트를 수행할 때 읽히며 새 프래그먼트가 그려질 때 쓰여집니다. 읽기는 `VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT` 단계에서 발생하고 쓰기는 `VK_PIPELINE_STAGE_LATE_FRAGMENT_TESTS_BIT`에서 발생합니다. 지정된 작업에 맞는 가장 이른 파이프라인 단계를 선택해야 하므로 깊이 첨부 파일로 사용해야 할 때 준비됩니다.

## 렌더 패스

이제 `createRenderPass`를 수정하여 깊이 첨부 파일을 포함시킬 것입니다. 먼저 `VkAttachmentDescription`을 지정하세요:

```c++
VkAttachmentDescription depthAttachment{};
depthAttachment.format = findDepthFormat();
depthAttachment.samples = VK_SAMPLE_COUNT_1_BIT;
depthAttachment.loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
depthAttachment.storeOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
depthAttachment.stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
depthAttachment.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
depthAttachment.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
depthAttachment.finalLayout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;
```

`format`은 깊이 이미지 자체와 동일해야 합니다. 이번에는 그림이 완료된 후에 깊이 데이터(`storeOp`)를 사용하지 않기 때문에 하드웨어가 추가 최적화를 수행할 수 있습니다. 색상 버퍼와 마찬가지로 이전 깊이 내용은 중요하지 않으므로 `initialLayout`으로 `VK_IMAGE_LAYOUT_UNDEFINED`을 사용할 수 있습니다.

```c++
VkAttachmentReference depthAttachmentRef{};
depthAttachmentRef.attachment = 1;
depthAttachmentRef.layout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;
```

첫 번째(유일한) 서브패스에 대한 첨부 파일 참조를 추가하세요:

```c++
VkSubpassDescription subpass{};
subpass.pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS;
subpass.colorAttachmentCount = 1;
subpass.pColorAttachments = &colorAttachmentRef;
subpass.pDepthStencilAttachment = &depthAttachmentRef;
```

색상 첨부 파일과 달리 서브패스는 하나의 깊이(+스텐실) 첨부 파일만 사용할 수 있습니다. 여러 버퍼에서 깊이 테스트를 수행하는 것은 별로 의미가 없습니다.

```c++
std::array<VkAttachmentDescription, 2> attachments = {colorAttachment, depthAttachment};
VkRenderPassCreateInfo renderPassInfo{};
renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO;
renderPassInfo.attachmentCount = static_cast<uint32_t>(attachments.size());
renderPassInfo.pAttachments = attachments.data();
renderPassInfo.subpassCount = 1;
renderPassInfo.pSubpasses = &subpass;
renderPassInfo.dependencyCount = 1;
renderPassInfo.pDependencies = &dependency;
```

다음으로, `VkSubpassDependency` 구조체를 업데이트하여 두 첨부 파일을 모두 참조하도록 하세요.

```c++
dependency.srcStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT | VK_PIPELINE_STAGE_LATE_FRAGMENT_TESTS_BIT;
dependency.srcAccessMask = VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT;
dependency.dstStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT | VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT;
dependency.dstAccessMask = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT | VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT;
```

마지막으로, 깊이 이미지의 전환과 로드 작업의 일부로 클리어되는 것 사이의 충돌이 없도록 서브패스 종속성을 확장해야 합니다. 깊이 이미지는 초기 프래그먼트 테스트 파이프라인 단계에서 처음 액세스되며 *클리어*하는 로드 작업이 있기 때문에 쓰기에 대한 액세스 마스크를 지정해야 합니다.

## 프레임버퍼

다음 단계는 프레임버퍼 생성을 수정하여 깊이 이미지를 깊이 첨부 파일에 바인딩하는 것입니다. `createFramebuffers`로 이동하여 깊이 이미지 뷰를 두 번째 첨부 파일로 지정하세요:

```c++
std::array<VkImageView, 2> attachments = {
    swapChainImageViews[i],
    depthImageView
};

VkFramebufferCreateInfo framebufferInfo{};
framebufferInfo.sType = VK_STRUCTURE_TYPE_FRAMEBUFFER_CREATE_INFO;
framebufferInfo.renderPass = renderPass;
framebufferInfo.attachmentCount = static_cast<uint32_t>(attachments.size());
framebufferInfo.pAttachments = attachments.data();
framebufferInfo.width = swapChainExtent.width;
framebufferInfo.height = swapChainExtent.height;
framebufferInfo.layers = 1;
```

색상 첨부 파일은 모든 스왑 체인 이미지마다 다르지만, 세마포어로 인해 동시에 하나의 서브패스만 실행되므로 동일한 깊이 이미지를 모두 사용할 수 있습니다.



또한 깊이 이미지 뷰가 실제로 생성된 후에 호출되도록 `createFramebuffers` 호출을 이동해야 합니다:

```c++
void initVulkan() {
    ...
    createDepthResources();
    createFramebuffers();
    ...
}
```

## 클리어 값

`VK_ATTACHMENT_LOAD_OP_CLEAR`을 사용하는 여러 첨부 파일이 있으므로 여러 클리어 값을 지정해야 합니다. `recordCommandBuffer`로 이동하여 `VkClearValue` 구조체의 배열을 생성하세요:

```c++
std::array<VkClearValue, 2> clearValues{};
clearValues[0].color = {{0.0f, 0.0f, 0.0f, 1.0f}};
clearValues[1].depthStencil = {1.0f, 0};

renderPassInfo.clearValueCount = static_cast<uint32_t>(clearValues.size());
renderPassInfo.pClearValues = clearValues.data();
```

Vulkan에서 깊이 버퍼의 깊이 범위는 `0.0`에서 `1.0`이며, 여기서 `1.0`은 멀리 보이는 평면에, `0.0`은 가까운 평면에 있습니다. 깊이 버퍼의 각 점의 초기 값은 가능한 가장 먼 깊이, 즉 `1.0`이어야 합니다.

`clearValues`의 순서는 첨부 파일의 순서와 동일해야 합니다.

## 깊이 및 스텐실 상태

깊이 첨부 파일이 이제 사용 준비가 되었지만, 깊이 테스트는 그래픽 파이프라인에서 활성화해야 합니다. 이는 `VkPipelineDepthStencilStateCreateInfo` 구조체를 통해 구성됩니다:

```c++
VkPipelineDepthStencilStateCreateInfo depthStencil{};
depthStencil.sType = VK_STRUCTURE_TYPE_PIPELINE_DEPTH_STENCIL_STATE_CREATE_INFO;
depthStencil.depthTestEnable = VK_TRUE;
depthStencil.depthWriteEnable = VK_TRUE;
```

`depthTestEnable` 필드는 새 프래그먼트의 깊이가 깊이 버퍼와 비교되어 버려질지 여부를 지정합니다. `depthWriteEnable` 필드는 깊이 테스트를 통과한 프래그먼트의 새 깊이가 실제로 깊이 버퍼에 기록되어야 하는지를 지정합니다.

```c++
depthStencil.depthCompareOp = VK_COMPARE_OP_LESS;
```

`depthCompareOp` 필드는 프래그먼트를 유지하거나 버리는 데 수행되는 비교를 지정합니다. 우리는 낮은 깊이 = 더 가까움의 규칙을 따르므로 새 프래그먼트의 깊이는 *더 적어야* 합니다.

```c++
depthStencil.depthBoundsTestEnable = VK_FALSE;
depthStencil.minDepthBounds = 0.0f; // 선택 사항
depthStencil.maxDepthBounds = 1.0f; // 선택 사항
```

`depthBoundsTestEnable`, `minDepthBounds`, `maxDepthBounds` 필드는 선택적 깊이 경계 테스트에 사용됩니다. 기본적으로 이 기능을 사용하면 지정된 깊이 범위 내에 있는 프래그먼트만 유지할 수 있습니다. 우리는 이 기능을 사용하지 않을 것입니다.

```c++
depthStencil.stencilTestEnable = VK_FALSE;
depthStencil.front = {}; // 선택 사항
depthStencil.back = {}; // 선택 사항
```

마지막 세 필드는 스텐실 버퍼 작업을 구성하는 데 사용되며, 이 튜토리얼에서는 사용하지 않을 것입니다. 이러한 작업을 사용하려면 깊이/스텐실 이미지의 형식이 스텐실 구성 요소를 포함하는지 확인해야 합니다.

```c++
pipelineInfo.pDepthStencilState = &depthStencil;
```

깊이 스텐실 상태를 참조하도록 `VkGraphicsPipelineCreateInfo` 구조체를 업데이트하세요. 렌더 패스에 깊이 스텐실 첨부 파일이 포함되어 있으면 항상 깊이 스텐실 상태를 지정해야 합니다.

이제 프로그램을 실행하면 기하학의 프래그먼트가 올바르게 정렬된 것을 볼 수 있습니다:

![](/images/depth_correct.png)

## 창 크기 변경 처리

창 크기가 변경될 때 깊이 버퍼의 해상도도 새 색상 첨부 파일 해상도와 일치하도록 변경해야 합니다. `recreateSwapChain` 함수를 확장하여 그 경우에 깊이 리소스를 다시 생성하세요:

```c++
void recreateSwapChain() {
    int width = 0, height = 0;
    while (width == 0 || height == 0) {
        glfwGetFramebufferSize(window, &width, &height);
        glfwWaitEvents();
    }

    vkDeviceWaitIdle(device);

    cleanupSwapChain();

    createSwapChain();
    createImageViews();
    createDepthResources();
    createFramebuffers();
}
```

정리 작업은 스왑 체인 정리 함수에서 수행해야 합니다:

```c++
void cleanupSwapChain() {
    vkDestroyImageView(device, depthImageView, nullptr);
    vkDestroyImage(device, depthImage, nullptr);
    vkFreeMemory(device, depthImageMemory, nullptr);

    ...
}
```

축하합니다, 이제 애플리케이션이 임의의 3D 기하학을 렌더링하고 올바르게 보이게 할 준비가 되었습니다. 다음 장에서 텍스처가 있는 모델을 그려보면서 이를 시험해볼 것입니다!

[C++ 코드](/code/27_depth_buffering.cpp) /
[버텍스 셰이더](/code/27_shader_depth.vert) /
[프래그먼트 셰이더](/code/27_shader_depth.frag)