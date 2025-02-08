# 이미지

## 소개

지금까지는 정점당 색상을 사용하여 기하학적 도형을 채색했지만, 이는 제한적인 접근 방식입니다. 이번 튜토리얼 파트에서는 텍스처 매핑을 구현하여 기하학적 도형을 더 흥미롭게 만들어 볼 것입니다. 이는 향후 3D 모델을 로드하고 그리는 데도 도움이 될 것입니다.

애플리케이션에 텍스처를 추가하는 작업은 다음과 같은 단계를 포함합니다:

- 장치 메모리로 백업된 이미지 객체 생성
- 이미지 파일에서 픽셀로 채우기
- 이미지 샘플러 생성
- 텍스처에서 색상을 샘플링하기 위해 결합된 이미지 샘플러 디스크립터 추가

이전에는 스왑 체인 확장을 통해 자동으로 생성된 이미지 객체를 사용했지만, 이번에는 직접 하나를 생성해야 합니다. 이미지 생성과 데이터로 채우는 과정은 버텍스 버퍼 생성과 유사합니다. 스테이징 리소스를 생성하고 픽셀 데이터로 채운 다음 렌더링에 사용할 최종 이미지 객체로 이 데이터를 복사할 것입니다. 스테이징 이미지를 생성하는 것도 가능하지만, Vulkan은 `VkBuffer`에서 이미지로 픽셀을 복사할 수 있게 하며, [일부 하드웨어에서는 이 API가 더 빠릅니다](https://developer.nvidia.com/vulkan-memory-management). 먼저 이 버퍼를 생성하고 픽셀 값으로 채운 다음, 픽셀을 복사할 이미지를 생성할 것입니다. 버퍼 생성과 마찬가지로, 메모리 요구 사항을 조회하고 장치 메모리를 할당하고 바인딩하는 과정을 포함합니다.

하지만 이미지 작업 시 추가적으로 고려해야 할 사항이 있습니다. 이미지는 메모리에서 픽셀이 어떻게 조직되어 있는지에 영향을 주는 다양한 *레이아웃*을 가질 수 있습니다. 예를 들어, 단순히 픽셀을 행별로 저장하는 것이 최선의 성능을 제공하지 않을 수 있습니다. 이미지에 대한 어떠한 작업을 수행할 때, 그 작업에 최적화된 레이아웃을 가지고 있는지 확인해야 합니다. 실제로 렌더 패스를 지정할 때 몇몇 레이아웃을 이미 보았습니다:

- `VK_IMAGE_LAYOUT_PRESENT_SRC_KHR`: 프레젠테이션에 최적화됨
- `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL`: 프래그먼트 셰이더에서 색상을 쓰는 데 최적화된 첨부 파일로 사용됨
- `VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL`: `vkCmdCopyImageToBuffer`와 같은 전송 작업의 소스로 사용될 때 최적화됨
- `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`: `vkCmdCopyBufferToImage`와 같은 전송 작업의 목적지로 사용될 때 최적화됨
- `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL`: 셰이더에서 샘플링할 때 최적화됨

이미지의 레이아웃을 전환하는 가장 일반적인 방법 중 하나는 *파이프라인 배리어*를 사용하는 것입니다. 파이프라인 배리어는 주로 리소스에 대한 접근을 동기화하는 데 사용되지만, 레이아웃 전환에도 사용할 수 있습니다. `VK_SHARING_MODE_EXCLUSIVE`를 사용할 때 큐 패밀리 소유권을 전송하는 데에도 사용할 수 있습니다.

## 이미지 라이브러리

이미지를 로드하기 위해 사용할 수 있는 많은 라이브러리가 있으며, BMP나 PPM과 같은 간단한 형식의 이미지를 로드하기 위한 코드를 직접 작성할 수도 있습니다. 이 튜토리얼에서는 [stb 컬렉션](https://github.com/nothings/stb)의 stb_image 라이브러리를 사용할 것입니다. 이 라이브러리의 장점은 모든 코드가 단일 파일에 포함되어 있어 복잡한 빌드 구성이 필요하지 않다는 것입니다. `stb_image.h`를 다운로드하여 GLFW와 GLM을 저장한 디렉토리와 같은 편리한 위치에 저장하세요. 인클루드 경로에 위치를 추가합니다.

**Visual Studio**

`stb_image.h`가 있는 디렉토리를 `Additional Include Directories` 경로에 추가합니다.

![](/images/include_dirs_stb.png)

**Makefile**

GCC의 인클루드 디렉토리에 `stb_image.h`가 있는 디렉토리를 추가합니다:

```text
VULKAN_SDK_PATH = /home/user/VulkanSDK/x.x.x.x/x86_64
STB_INCLUDE_PATH = /home/user/libraries/stb

...

CFLAGS = -std=c++17 -I$(VULKAN_SDK_PATH)/include -I$(STB_INCLUDE_PATH)
```

## 이미지 로딩

이미지 라이브러리를 다음과 같이 포함합니다:

```c++
#define STB_IMAGE_IMPLEMENTATION
#include <stb_image.h>
```

기본적으로 헤더는 함수의 프로토타입만 정의합니다. 함수 본문을 포함하려면 한 코드 파일에서 `STB_IMAGE_IMPLEMENTATION` 정의와 함께 헤더를 포함해야 하며, 그렇지 않으면 링크 오류가 발생합니다.

```c++
void initVulkan() {
    ...
    createCommandPool();
    createTextureImage();
    createVertexBuffer();
    ...
}

...

void createTextureImage() {

}
```

`createCommandPool` 후에 호출되어야 하므로 새로운 함수 `createTextureImage`를 만들어 이미지를 로드하고 Vulkan 이미지 객체로 업로드합니다.

`shaders` 디렉토리 옆에 텍스처 이미지를 저장할 `textures` 새 디

렉토리를 만듭니다. 그 디렉토리에서 `texture.jpg`라는 이미지를 로드할 것입니다. 512 x 512 픽셀로 크기를 조절한 다음 [CC0 라이선스 이미지](https://pixabay.com/en/statue-sculpture-fig-historically-1275469/)를 사용했지만, 원하는 이미지를 자유롭게 선택할 수 있습니다. 라이브러리는 JPEG, PNG, BMP, GIF와 같은 대부분의 일반 이미지 파일 형식을 지원합니다.

![](/images/texture.jpg)

이 라이브러리를 사용하여 이미지를 로드하는 것은 매우 쉽습니다:

```c++
void createTextureImage() {
    int texWidth, texHeight, texChannels;
    stbi_uc* pixels = stbi_load("textures/texture.jpg", &texWidth, &texHeight, &texChannels, STBI_rgb_alpha);
    VkDeviceSize imageSize = texWidth * texHeight * 4;

    if (!pixels) {
        throw std::runtime_error("failed to load texture image!");
    }
}
```

`stbi_load` 함수는 파일 경로와 로드할 채널 수를 인수로 받습니다. `STBI_rgb_alpha` 값은 이미지가 알파 채널을 갖고 있지 않더라도 알파 채널을 갖도록 강제합니다. 이는 일관성을 위해 좋습니다. 중간 세 매개변수는 이미지의 너비, 높이 및 실제 채널 수를 출력합니다. 반환되는 포인터는 픽셀 값 배열의 첫 번째 요소입니다. 픽셀은 `STBI_rgb_alpha`의 경우 픽셀당 4바이트를 사용하여 행별로 배치되어 총 `texWidth * texHeight * 4` 값이 됩니다.

## 스테이징 버퍼

이제 호스트 가시 메모리에서 버퍼를 생성하여 `vkMapMemory`를 사용하고 그 안에 픽셀을 복사할 수 있습니다. `createTextureImage` 함수에 이 임시 버퍼에 대한 변수를 추가하세요:

```c++
VkBuffer stagingBuffer;
VkDeviceMemory stagingBufferMemory;
```

버퍼는 호스트 가시 메모리에 있어야 하므로 매핑할 수 있으며, 나중에 이미지로 복사할 수 있도록 전송 소스로 사용 가능해야 합니다:

```c++
createBuffer(imageSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, stagingBuffer, stagingBufferMemory);
```

그런 다음 이미지 로딩 라이브러리에서 얻은 픽셀 값을 버퍼에 직접 복사할 수 있습니다:

```c++
void* data;
vkMapMemory(device, stagingBufferMemory, 0, imageSize, 0, &data);
    memcpy(data, pixels, static_cast<size_t>(imageSize));
vkUnmapMemory(device, stagingBufferMemory);
```

원본 픽셀 배열을 정리하는 것을 잊지 마세요:

```c++
stbi_image_free(pixels);
```

## 텍스처 이미지

셰이더에서 픽셀 값을 버퍼로 설정하여 액세스할 수 있지만, Vulkan에서는 이 목적을 위해 이미지 객체를 사용하는 것이 더 낫습니다. 이미지 객체는 2D 좌표를 사용할 수 있게 하여 색상을 검색하는 것을 더 쉽고 빠르게 할 수 있습니다. 이 지점부터는 픽셀이 아닌 텍셀이라는 이름을 사용할 것입니다. 다음과 같은 새 클래스 멤버를 추가하세요:

```c++
VkImage textureImage;
VkDeviceMemory textureImageMemory;
```

이미지의 매개변수는 `VkImageCreateInfo` 구조체에서 지정됩니다:

```c++
VkImageCreateInfo imageInfo{};
imageInfo.sType = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO;
imageInfo.imageType = VK_IMAGE_TYPE_2D;
imageInfo.extent.width = static_cast<uint32_t>(texWidth);
imageInfo.extent.height = static_cast<uint32_t>(texHeight);
imageInfo.extent.depth = 1;
imageInfo.mipLevels = 1;
imageInfo.arrayLayers = 1;
```

`imageType` 필드에서 지정된 이미지 유형은 텍셀이 이미지에서 어떻게 주소 지정될지를 알려줍니다. 1D, 2D, 3D 이미지를 만들 수 있습니다. 일차원 이미지는 데이터 배열이나 그라디언트를 저장하는 데 사용할 수 있고, 이차원 이미지는 주로 텍스처에 사용되며, 삼차원 이미지는 예를 들어 복셀 볼륨을 저장하는 데 사용할 수 있습니다. `extent` 필드는 이미지의 치수를 지정합니다. 즉, 각 축에 몇 개의 텍셀이 있는지를 나타냅니다. 그래서 `depth`는 `0`이 아니라 `1`이어야 합니다. 우리의 텍스처는 배열이 아니며 지금은 미핑을 사용하지 않을 것입니다.

```c++
imageInfo.format = VK_FORMAT_R8G8B8A8_SRGB;
```

Vulkan은 많은 가능한 이미지 형식을 지원하지만, 버퍼의 픽셀과 동일한 형식을 사용해야 합니다. 그렇지 않으면 복사 작업이 실패합니다.

```c++
imageInfo.tiling = VK_IMAGE_TILING_OPTIMAL;
```

`tiling` 필드는 두 가지 값 중 하나를 가질 수 있습니다:

- `VK_IMAGE_TILING_LINEAR`: 픽셀 배열처럼 텍셀이 행별로 늘어섭니다
- `VK_IMAGE_TILING_OPTIMAL`: 텍셀이 구현에 따라 최적의 접근을 위해 정의된 순서로 늘어섭니다

이미지의 레이아웃은 나중에 변경할 수 없습니다. 메모리의 텍셀에 직접 접근하려면 `VK_IMAGE_TILING_LINEAR`를 사용해야 합니다. 우리는 스테이징 버퍼 대신 스테이징 이미지를 사용할 것이므로 이것이 필요하지 않습니다. 셰이더에서 효율적으로 접근하기 위해 `VK_IMAGE_TILING_OPTIMAL`을 사용할 것입니다.

```c++
imageInfo.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
```

이미지의 `initialLayout`에는 두 가지 가능한 값이 있습니다:

- `VK_IMAGE_LAYOUT_UNDEFINED`: GPU에서 사용할 수 없으며 첫 번째 전환은 텍셀을 폐기

합니다.
- `VK_IMAGE_LAYOUT_PREINITIALIZED`: GPU에서 사용할 수 없지만 첫 번째 전환은 텍셀을 보존합니다.

텍셀을 보존할 필요가 있는 몇 가지 상황이 있습니다. 하나의 예는 `VK_IMAGE_TILING_LINEAR` 레이아웃과 함께 이미지를 스테이징 이미지로 사용하려는 경우일 수 있습니다. 그 경우 텍셀 데이터를 업로드한 다음 전송 소스로 전환하면서 데이터를 유지하고 싶을 것입니다. 그러나 우리의 경우는 먼저 이미지를 전송 목적지로 전환한 다음 버퍼 객체에서 텍셀 데이터를 복사할 것이므로 이 속성이 필요하지 않고 안전하게 `VK_IMAGE_LAYOUT_UNDEFINED`을 사용할 수 있습니다.

```c++
imageInfo.usage = VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT;
```

`usage` 필드는 버퍼 생성 때와 같은 의미를 갖습니다. 이미지는 버퍼 복사의 목적지로 사용될 것이므로 전송 목적지로 설정해야 합니다. 또한 셰이더에서 메시를 색칠하기 위해 이미지에서 색상을 샘플링하려면 사용 용도에 `VK_IMAGE_USAGE_SAMPLED_BIT`를 포함해야 합니다.

```c++
imageInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;
```

이미지는 하나의 큐 패밀리만 사용할 것입니다: 그래픽(따라서 전송 작업도 포함) 작업을 지원하는 큐 패밀리.

```c++
imageInfo.samples = VK_SAMPLE_COUNT_1_BIT;
imageInfo.flags = 0; // Optional
```

`samples` 플래그는 멀티샘플링과 관련이 있습니다. 이는 이미지가 첨부 파일로 사용될 때만 관련이 있으므로 한 샘플로 고정하세요. 이미지에는 희소 이미지와 관련된 몇 가지 선택적 플래그가 있습니다. 희소 이미지는 실제로 메모리가 백업되는 특정 영역만 있는 이미지입니다. 예를 들어, 복셀 지형을 사용하는 경우 "공기" 값을 저장하기 위해 메모리를 할당하는 것을 피하기 위해 이를 사용할 수 있습니다. 이 튜토리얼에서는 사용하지 않을 것이므로 기본값 `0`을 그대로 사용하세요.

```c++
if (vkCreateImage(device, &imageInfo, nullptr, &textureImage) != VK_SUCCESS) {
    throw std::runtime_error("failed to create image!");
}
```

이미지는 `vkCreateImage`를 사용하여 생성됩니다. 특별히 주목할 만한 매개변수는 없습니다. `VK_FORMAT_R8G8B8A8_SRGB` 형식이 그래픽 하드웨어에서 지원되지 않을 수 있습니다. 받아들일 수 있는 대안 목록을 가지고 있어야 하며 지원되는 최선의 형식을 선택해야 합니다. 그러나 이 특정 형식에 대한 지원은 매우 널리 퍼져 있어 이 단계를 생략할 것입니다. 다른 형식을 사용하면 성가신 변환도 필요합니다. 깊이 버퍼 장에서 이러한 시스템을 구현할 때 이에 대해 다시 살펴볼 것입니다.

```c++
VkMemoryRequirements memRequirements;
vkGetImageMemoryRequirements(device, textureImage, &memRequirements);

VkMemoryAllocateInfo allocInfo{};
allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
allocInfo.allocationSize = memRequirements.size;
allocInfo.memoryTypeIndex = findMemoryType(memRequirements.memoryTypeBits, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT);

if (vkAllocateMemory(device, &allocInfo, nullptr, &textureImageMemory) != VK_SUCCESS) {
    throw std::runtime_error("failed to allocate image memory!");
}

vkBindImageMemory(device, textureImage, textureImageMemory, 0);
```

이미지에 메모리를 할당하는 것은 버퍼에 메모리를 할당하는 것과 정확히 같은 방식으로 작동합니다. `vkGetBufferMemoryRequirements` 대신 `vkGetImageMemoryRequirements`를 사용하고, `vkBindBufferMemory` 대신 `vkBindImageMemory`를 사용하세요.

이 함수는 이미 상당히 크고 나중에 더 많은 이미지를 생성할 것이므로 버퍼와 같이 이미지 생성을 `createImage` 함수로 추상화하는 것이 좋습니다. 함수를 생성하고 이미지 객체 생성 및 메모리 할당을 이동하세요:

```c++
void createImage(uint32_t width, uint32_t height, VkFormat format, VkImageTiling tiling, VkImageUsageFlags usage, VkMemoryPropertyFlags properties, VkImage& image, VkDeviceMemory& imageMemory) {
    VkImageCreateInfo imageInfo{};
    imageInfo.sType = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO;
    imageInfo.imageType = VK_IMAGE_TYPE_2D;
    imageInfo.extent.width = width;
    imageInfo.extent.height = height;
    imageInfo.extent.depth = 1;
    imageInfo.mipLevels = 1;
    imageInfo.arrayLayers = 1;
    imageInfo.format = format;
    imageInfo.tiling = tiling;
    imageInfo.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
    imageInfo.usage = usage;
    imageInfo.samples = VK_SAMPLE_COUNT_1_BIT;
    imageInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;

    if (vkCreateImage(device, &imageInfo, nullptr, &image) != VK_SUCCESS) {
        throw std::runtime_error("failed to create image!");
    }

    VkMemoryRequirements memRequirements;
    vkGetImageMemoryRequirements(device, image, &memRequirements);

    VkMemoryAllocateInfo allocInfo{};
    allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
    allocInfo.allocationSize = memRequirements.size;
    allocInfo.memoryTypeIndex = findMemoryType(memRequirements.memoryTypeBits, properties);

    if (vkAllocateMemory(device, &allocInfo, nullptr, &imageMemory) != VK_SUCCESS) {
        throw std::runtime_error("failed to allocate image memory!");
    }

    vkBindImageMemory(device, image, imageMemory, 0);
}
```

너비, 높이, 형식, 타일링 모드, 사용 용도 및 메모리 속성을 매개변수로 사용했습니다. 이는 이 튜토리얼을 통해 생성할 이미지가 다양할 것이기 때문입니다.

`createTextureImage` 함수는 이제 간소화할 수 있습니다:

```c++
void createTextureImage() {
    int texWidth, texHeight, texChannels;
    stbi_uc* pixels = stbi_load("textures/texture.jpg", &texWidth, &texHeight, &texChannels, STBI_rgb_alpha);
    VkDeviceSize imageSize = texWidth * texHeight * 4;

    if (!pixels) {
        throw std::runtime_error("failed to load texture image!");
    }

    VkBuffer stagingBuffer;
    VkDeviceMemory stagingBufferMemory;
    createBuffer(imageSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, stagingBuffer, stagingBufferMemory);

    void* data;
    vkMapMemory(device, stagingBufferMemory, 0, imageSize, 0, &data);
        memcpy(data, pixels, static_cast<size_t>(imageSize));
    vkUnmapMemory(device, stagingBufferMemory);

    stbi_image_free(pixels);

    createImage(texWidth, texHeight, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, textureImage, textureImageMemory);
}
```

## 레이아웃 전환

이제 명령 버퍼를 다시 기록하고 실행하는 작업을 수행할 것이므로, 이 로직을 하나 또는 두 개의 도우미 함수로 이동할 좋은 시기입니다:

```c++
VkCommandBuffer beginSingleTimeCommands() {
    VkCommandBufferAllocateInfo allocInfo{};


    allocInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
    allocInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
    allocInfo.commandPool = commandPool;
    allocInfo.commandBufferCount = 1;

    VkCommandBuffer commandBuffer;
    vkAllocateCommandBuffers(device, &allocInfo, &commandBuffer);

    VkCommandBufferBeginInfo beginInfo{};
    beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
    beginInfo.flags = VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT;

    vkBeginCommandBuffer(commandBuffer, &beginInfo);

    return commandBuffer;
}

void endSingleTimeCommands(VkCommandBuffer commandBuffer) {
    vkEndCommandBuffer(commandBuffer);

    VkSubmitInfo submitInfo{};
    submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;
    submitInfo.commandBufferCount = 1;
    submitInfo.pCommandBuffers = &commandBuffer;

    vkQueueSubmit(graphicsQueue, 1, &submitInfo, VK_NULL_HANDLE);
    vkQueueWaitIdle(graphicsQueue);

    vkFreeCommandBuffers(device, commandPool, 1, &commandBuffer);
}
```

`copyBuffer` 함수에서 기존 코드를 기반으로 함수를 작성했습니다. 이제 이 함수를 간소화할 수 있습니다:

```c++
void copyBuffer(VkBuffer srcBuffer, VkBuffer dstBuffer, VkDeviceSize size) {
    VkCommandBuffer commandBuffer = beginSingleTimeCommands();

    VkBufferCopy copyRegion{};
    copyRegion.size = size;
    vkCmdCopyBuffer(commandBuffer, srcBuffer, dstBuffer, 1, &copyRegion);

    endSingleTimeCommands(commandBuffer);
}
```

버퍼를 사용하고 있다면, 이제 `vkCmdCopyBufferToImage`를 기록하고 실행하여 작업을 완료할 수 있는 함수를 작성할 수 있습니다. 하지만 이 명령은 먼저 이미지가 올바른 레이아웃에 있어야 합니다. 레이아웃 전환을 처리하는 새 함수를 작성하세요:

```c++
void transitionImageLayout(VkImage image, VkFormat format, VkImageLayout oldLayout, VkImageLayout newLayout) {
    VkCommandBuffer commandBuffer = beginSingleTimeCommands();

    endSingleTimeCommands(commandBuffer);
}
```

레이아웃 전환을 수행하는 가장 일반적인 방법 중 하나는 *이미지 메모리 배리어*를 사용하는 것입니다. 파이프라인 배리어는 주로 리소스에 대한 접근을 동기화하는 데 사용됩니다. 예를 들어, 읽기 전에 버퍼에 대한 쓰기가 완료되었는지 확인하는 것과 같습니다. 하지만 이미지 레이아웃을 전환하고 `VK_SHARING_MODE_EXCLUSIVE`를 사용할 때 큐 패밀리 소유권을 전송하는 데에도 사용할 수 있습니다. 버퍼에 대해 동일한 작업을 수행하는 *버퍼 메모리 배리어*도 있습니다.

```c++
VkImageMemoryBarrier barrier{};
barrier.sType = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER;
barrier.oldLayout = oldLayout;
barrier.newLayout = newLayout;
```

첫 두 필드는 레이아웃 전환을 지정합니다. 이미지의 기존 내용을 신경 쓰지 않는 경우 `oldLayout`로 `VK_IMAGE_LAYOUT_UNDEFINED`를 사용할 수 있습니다.

```c++
barrier.srcQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
barrier.dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
```

배리어를 사용하여 큐 패밀리 소유권을 전송하는 경우, 이 두 필드는 큐 패밀리의 인덱스여야 합니다. 이 작업을 수행하지 않으려면 `VK_QUEUE_FAMILY_IGNORED`로 설정해야 합니다(기본값이 아닙니다).

```c++
barrier.image = image;
barrier.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
barrier.subresourceRange.baseMipLevel = 0;
barrier.subresourceRange.levelCount = 1;
barrier.subresourceRange.baseArrayLayer = 0;
barrier.subresourceRange.layerCount = 1;
```

`image`와 `subresourceRange`는 영향을 받는 이미지와 이미지의 특정 부분을 지정합니다. 우리의 이미지는 배열이 아니며 미핑 레벨이 없으므로 하나의 레벨과 레이어만 지정됩니다.

```c++
barrier.srcAccessMask

 = 0; // TODO
barrier.dstAccessMask = 0; // TODO
```

배리어는 주로 동기화 목적으로 사용되므로, 배리어 전에 리소스와 관련된 작업 유형과 배리어가 완료된 후 리소스와 관련된 작업 유형을 지정해야 합니다. `vkQueueWaitIdle`을 수동으로 사용하여 동기화하더라도 이 작업을 수행해야 합니다. 올바른 값은 이전 및 새 레이아웃에 따라 다릅니다. 어떤 전환을 사용할지 결정되면 이 부분으로 돌아와서 작업하겠습니다.

```c++
vkCmdPipelineBarrier(
    commandBuffer,
    0 /* TODO */, 0 /* TODO */,
    0,
    0, nullptr,
    0, nullptr,
    1, &barrier
);
```

모든 유형의 파이프라인 배리어는 동일한 함수를 사용하여 제출됩니다. 명령 버퍼 다음의 첫 번째 매개변수는 배리어 전에 수행되는 작업이 발생하는 파이프라인 단계를 지정합니다. 두 번째 매개변수는 배리어에서 작업이 대기하는 파이프라인 단계를 지정합니다. 배리어 전후에 지정할 수 있는 파이프라인 단계는 리소스를 사용하는 방법에 따라 다릅니다. [명세](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap7.html#synchronization-access-types-supported)에서 허용되는 값 목록을 확인할 수 있습니다. 예를 들어, 배리어 후에 유니폼을 읽으려면 `VK_ACCESS_UNIFORM_READ_BIT` 사용을 지정하고 유니폼을 읽을 가장 이른 셰이더 단계를 파이프라인 단계로 지정해야 합니다. 예를 들어 `VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT`과 같습니다. 사용 유형에 맞지 않는 파이프라인 단계를 지정하면 유효성 검사 계층에서 경고합니다.

세 번째 매개변수는 `0` 또는 `VK_DEPENDENCY_BY_REGION_BIT`입니다. 후자는 배리어를 지역별 조건으로 설정합니다. 즉, 구현이 지금까지 작성된 리소스 부분에서 이미 읽기를 시작할 수 있도록 허용됩니다.

마지막 세 쌍의 매개변수는 파이프라인 배리어의 세 가지 유형을 참조하는 배열을 참조합니다: 메모리 배리어, 버퍼 메모리 배리어 및 여기서 사용하는 이미지 메모리 배리어입니다. 아직 `VkFormat` 매개변수를 사용하지 않았지만, 깊이 버퍼 장에서 특별한 전환을 위해 사용할 것입니다.

## 버퍼에서 이미지로 복사

`createTextureImage`로 돌아가기 전에 또 다른 도우미 함수를 작성할 것입니다: `copyBufferToImage`:

```c++
void copyBufferToImage(VkBuffer buffer, VkImage image, uint32_t width, uint32_t height) {
    VkCommandBuffer commandBuffer = beginSingleTimeCommands();

    endSingleTimeCommands(commandBuffer);
}
```

버퍼의 어느 부분이 이미지의 어느 부분으로 복사될지는 `VkBufferImageCopy` 구조체를 사용하여 지정해야 합니다:

```c++
VkBufferImageCopy region{};
region.bufferOffset = 0;
region.bufferRowLength = 0;
region.bufferImageHeight = 0;

region.imageSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
region.imageSubresource.mipLevel = 0;
region.imageSubresource.baseArrayLayer = 0;
region.imageSubresource.layerCount = 1;

region.imageOffset = {0, 0, 0};
region.imageExtent = {
    width,
    height,
    1
};
```

대부분의 필드는 자명합니다. `bufferOffset`은 버퍼에서 픽셀 값이 시작되는 바이트 오프셋을 지정합니다. `bufferRowLength`와 `bufferImageHeight` 필드는 메모리에 픽셀이 어떻게 배치되어 있는지를 지정합니다. 예를 들어, 이미지의 행 사이에 패딩 바이트가 있을 수 있습니다. `0`을 지정하면 픽셀이 우리의 경우처럼 단순히 밀집되어 있다는 것을 나타냅니다. `imageSubresource`, `imageOffset` 및 `imageExtent` 필드는 픽셀을 복사할 이미지의 부분을 지정합니다.

버퍼에서 이미지로의 복사 작업은 `vkCmdCopyBufferToImage` 함수를 사용하여 큐에 추가됩니다:

```c++
vkCmdCopyBufferToImage(
    commandBuffer,
    buffer,
    image,
    VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL,
    1,
    &region
);
```

네 번째 매개변수는 이미지가 현재 사용 중인 레이아웃을 나타냅니다. 여기서는 이미지가 복사 작업을 수행하기에 최적의 레이아웃인 `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`로 전환되었다고 가정합니다. 지금은 버퍼에서 이미지 전체로 픽셀을 복사하는 하나의 청크만 복사하고 있지만, `VkBufferImageCopy`의 배열을 지정하여 이 버퍼에서 이미지로 한 번의 작업으로 많은 다양한 복사를 수행할 수 있습니다.

## 텍스처 이미지 준비

이제 필요한 모든 도구를 갖추었으므로 `createTextureImage` 함수를 완성할 준비가 되었습니다. 마지막으로 한 작업은 텍스처 이미지를 생성하는 것이었습니다. 다음 단계는 스테이징 버퍼를 텍스처 이미지로 복사하는 것입니다. 이 작업은 두 단계를 포함합니다:

- 텍스처 이미지를 `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`로 전환
- 버퍼에서 이미지로 복사 작업 실행

이제 막 작성한 함수를 사용하면 쉽게 수행할 수 있습니다:

```c++
transitionImageLayout(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL);
copyBufferToImage(stagingBuffer, textureImage, static_cast<uint32_t>(texWidth), static_cast<uint32_t>(texHeight));
```

이미지는 `VK_IMAGE_LAYOUT_UNDEFINED` 레이아웃으로 생성되었으므로 `textureImage`를 전환할 때 이전 레이아웃으로 지정해야 합니다. 복사 작업을 수행하기 전에 이미지의 기존 내용을 신경 쓰지 않으므로 이렇게 할 수 있습니다.

셰이더에서 텍스처를 샘플링하기 시작하려면 마지막 전환을 수행하여 셰이더 액세스를 위해 준비해야 합니다:

```c++
transitionImageLayout(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL);
```

## 전환 배리어 마스크

지금 유효성 검사 계층을 활성화한 상태에서 애플리케이션을 실행하면, `transitionImageLayout`에서 접근 마스크와 파이프라인 단계가 잘

못되었다는 것을 알려줍니다. 이전에 설정한 레이아웃에 따라 이러한 값을 설정해야 합니다.

다루어야 할 두 가지 전환이 있습니다:

- 정의되지 않음 → 전송 대상: 대기할 필요 없는 전송 쓰기
- 전송 대상 → 셰이더 읽기: 셰이더 읽기는 전송 쓰기를 기다려야 하며, 특히 텍스처를 사용할 프래그먼트 셰이더에서 이를 사용해야 합니다

이러한 규칙은 다음 접근 마스크와 파이프라인 단계를 사용하여 지정됩니다:

```c++
VkPipelineStageFlags sourceStage;
VkPipelineStageFlags destinationStage;

if (oldLayout == VK_IMAGE_LAYOUT_UNDEFINED && newLayout == VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL) {
    barrier.srcAccessMask = 0;
    barrier.dstAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;

    sourceStage = VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT;
    destinationStage = VK_PIPELINE_STAGE_TRANSFER_BIT;
} else if (oldLayout == VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL && newLayout == VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL) {
    barrier.srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
    barrier.dstAccessMask = VK_ACCESS_SHADER_READ_BIT;

    sourceStage = VK_PIPELINE_STAGE_TRANSFER_BIT;
    destinationStage = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
} else {
    throw std::invalid_argument("unsupported layout transition!");
}

vkCmdPipelineBarrier(
    commandBuffer,
    sourceStage, destinationStage,
    0,
    0, nullptr,
    0, nullptr,
    1, &barrier
);
```

알 수 있듯이, 전송 쓰기는 파이프라인 전송 단계에서 발생해야 합니다. 쓰기가 어떤 것에도 기다릴 필요가 없으므로, 배리어 사전 작업에 대해 빈 접근 마스크와 가능한 가장 이른 파이프라인 단계인 `VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT`를 지정할 수 있습니다. `VK_PIPELINE_STAGE_TRANSFER_BIT`는 그래픽 및 컴퓨트 파이프라인 내의 *실제* 단계가 아닌 의사(pseudo) 단계입니다. 전송이 일어나는 곳입니다. 자세한 내용과 다른 예시에 대한 의사 단계는 [문서](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap7.html#VkPipelineStageFlagBits)를 참조하세요.

이미지는 동일한 파이프라인 단계에서 쓰여지고 이후에 프래그먼트 셰이더에서 읽힐 것입니다. 이것이 우리가 프래그먼트 셰이더 파이프라인 단계에서 셰이더 읽기 접근을 지정하는 이유입니다.

더 많은 전환을 처리해야 할 경우 함수를 확장할 것입니다. 애플리케이션은 이제 성공적으로 실행되어야 하지만 물론 아직 시각적 변화는 없습니다.

한 가지 주목할 점은 명령 버퍼 제출은 시작 시 암시적 `VK_ACCESS_HOST_WRITE_BIT` 동기화를 초래한다는 것입니다. `transitionImageLayout` 함수는 단일 명령으로 명령 버퍼를 실행하므로, 레이아웃 전환에서 `VK_ACCESS_HOST_WRITE_BIT` 종속성이 필요한 경우 이 암시적 동기화를 사용하고 `srcAccessMask`를 `0`으로 설정할 수 있습니다. 이에 대해 명시적으로 설정하고 싶은지 아니면 이러한 OpenGL과 같은 "숨겨진" 작업에 의존하고 싶지 않은지 여부는 귀하에게 달려 있습니다.

사실, 모든 작업을 지원하는 특별한 유형의 이미지 레이아웃이 있습니다: `VK_IMAGE_LAYOUT_GENERAL`. 물론 문제는 이 레이아웃이 어떤 작업에 대해서도 최상의 성능을 제공하지 않는다는 것입니다. 이미지를 입력 및 출력으로 사용하거나 이미지가 사전 초기화된 레이아웃을 벗어난 후 이미지를 읽을 필요가 있는 일부 특별한 경우에 필요합니다.

지금까지 도우미 함수가 제출하는 모든 명령은 큐가 유휴 상태가 될 때까지 기다리는 방식으로 동기적으로 설정되었습니다. 실제 애플리케이션에서는 이러한 작업을 단일 명령 버퍼에 결합하고 비동기적으로 실행하여 처리량을 높이는 것이 좋습니다. 특히 `createTextureImage` 함수에서 전환과 복사를 수행할 때 이를 시도해 보세요. `setupCommandBuffer`를 만들어 도우미 함수가 명령을 기록하도록 하고, 지금까지 기록된 명령을 실행하는 `flushSetupCommands`를 추가하세요. 텍스처 매핑이 작동한 후에 이를 시도하는 것이 좋습니다. 텍스처 리소스가 여전히 올바르게 설정되어 있는지 확인할 수 있습니다.

## 정리

`createTextureImage` 함수를 마무리하고 끝에서 스테이징 버퍼와 그 메모리를 정리하세요:

```c++
    transitionImageLayout(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL);

    vkDestroyBuffer(device, stagingBuffer, nullptr);
    vkFreeMemory(device, stagingBufferMemory, nullptr);
}
```

메인 텍스처 이미지는 프로그램이 끝날 때까지 사용됩니다:

```c++
void cleanup() {
    cleanupSwapChain();

    vkDestroyImage(device, textureImage, nullptr);
    vkFreeMemory(device, textureImageMemory, nullptr);

    ...
}
```

이제 이미지에 텍스처가 포함되어 있지만, 그래픽 파이프라인에서 액세스할 수 있는 방법이 필요합니다. 다음 장에서 이 작업을 수행할 것입니다.

[C++ 코드](/code/24_texture_image.cpp) /
[버텍스 셰이더](/code/22_shader_ubo.vert) /
[프래그먼트 셰이더](/code/22_shader_ubo.frag)
