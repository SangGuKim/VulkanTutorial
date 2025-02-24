# 밉맵(mipmap) 생성

## 소개
이제 프로그램은 3D 모델을 불러오고 렌더링할 수 있습니다. 이 장에서는 밉맵 생성이라는 기능을 추가할 것입니다. 밉맵은 게임과 렌더링 소프트웨어에서 널리 사용되며, Vulkan은 밉맵이 생성되는 방식을 완전히 제어할 수 있게 해줍니다.

밉맵은 이미지의 사전 계산된 축소 버전입니다. 각 새 이미지는 이전 이미지의 너비와 높이의 절반입니다. 밉맵은 *세부 수준* 또는 *LOD*의 한 형태로 사용됩니다. 카메라에서 멀리 떨어진 객체는 더 작은 밉맵 이미지에서 텍스처를 샘플링합니다. 더 작은 이미지를 사용하면 렌더링 속도가 향상되고 [모아레 패턴](https://en.wikipedia.org/wiki/Moir%C3%A9_pattern)과 같은 아티팩트를 방지할 수 있습니다. 밉맵이 어떻게 보이는지 예를 들어보겠습니다:

![](/images/mipmaps_example.jpg)

## 이미지 생성

Vulkan에서 각 밉맵 이미지는 `VkImage`의 다른 *밉맵 레벨*에 저장됩니다. 밉맵 레벨 0은 원본 이미지이며, 레벨 0 이후의 밉맵 레벨은 일반적으로 *밉맵 체인*이라고 합니다.

`VkImage`가 생성될 때 밉맵 레벨의 수가 지정됩니다. 지금까지 우리는 이 값을 항상 1로 설정했습니다. 이미지의 치수에서 밉맵 레벨의 수를 계산할 필요가 있습니다. 먼저 이 수를 저장할 클래스 멤버를 추가하세요:

```c++
...
uint32_t mipLevels;
VkImage textureImage;
...
```

`createTextureImage`에서 텍스처를 불러온 후 `mipLevels` 값을 찾을 수 있습니다:

```c++
int texWidth, texHeight, texChannels;
stbi_uc* pixels = stbi_load(TEXTURE_PATH.c_str(), &texWidth, &texHeight, &texChannels, STBI_rgb_alpha);
...
mipLevels = static_cast<uint32_t>(std::floor(std::log2(std::max(texWidth, texHeight)))) + 1;
```

이것은 밉맵 체인의 레벨 수를 계산합니다. `max` 함수는 가장 큰 치수를 선택합니다. `log2` 함수는 그 치수를 2로 몇 번 나눌 수 있는지 계산합니다. `floor` 함수는 가장 큰 치수가 2의 거듭제곱이 아닌 경우를 처리합니다. 원본 이미지에 밉맵 레벨이 있도록 1을 더합니다.

이 값을 사용하려면 `createImage`, `createImageView`, `transitionImageLayout` 함수를 변경하여 밉맵 레벨의 수를 지정할 수 있도록 해야 합니다. 함수에 `mipLevels` 매개변수를 추가하세요:

```c++
void createImage(uint32_t width, uint32_t height, uint32_t mipLevels, VkFormat format, VkImageTiling tiling, VkImageUsageFlags usage, VkMemoryPropertyFlags properties, VkImage& image, VkDeviceMemory& imageMemory) {
    ...
    imageInfo.mipLevels = mipLevels;
    ...
}
```
```c++
VkImageView createImageView(VkImage image, VkFormat format, VkImageAspectFlags aspectFlags, uint32_t mipLevels) {
    ...
    viewInfo.subresourceRange.levelCount = mipLevels;
    ...
```
```c++
void transitionImageLayout(VkImage image, VkFormat format, VkImageLayout oldLayout, VkImageLayout newLayout, uint32_t mipLevels) {
    ...
    barrier.subresourceRange.levelCount = mipLevels;
    ...
```

이 함수들의 모든 호출을 업데이트하여 올바른 값을 사용하세요:

```c++
createImage(swapChainExtent.width, swapChainExtent.height, 1, depthFormat, VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, depthImage, depthImageMemory);
...
createImage(texWidth, texHeight, mipLevels, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, textureImage, textureImageMemory);
```
```c++
swapChainImageViews[i] = createImageView(swapChainImages[i], swapChainImageFormat, VK_IMAGE_ASPECT_COLOR_BIT, 1);
...
depthImageView = createImageView(depthImage, depthFormat, VK_IMAGE_ASPECT_DEPTH_BIT, 1);
...
textureImageView = createImageView(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_ASPECT_COLOR_BIT, mipLevels);
```
```c++
transitionImageLayout(depthImage, depthFormat, VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL, 1);
...
transitionImageLayout(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, mipLevels);
```

## 밉맵 생성

우리의 텍스처 이미지에는 이제 여러 밉맵 레벨이 있지만, 스테이징 버퍼는 밉맵 레벨 0만 채울 수 있습니다. 다른 레벨은 여전히 정의되지 않았습니다. 이 레벨들을 채우려면 가지고 있는 단일 레벨에서 데이터를 생성해야 합니다. `vkCmdBlitImage` 명령을 사용할 것입니다. 이 명령은 복사, 크기 조정 및 필터링 작업을 수행합니다. 우리는 이 명령을 여러 번 호출하여 텍스처 이미지의 각 레벨에 데이터를 *blit*할 것입니다.

`vkCmdBlitImage`는 전송 작업으로 간주되므로 Vulkan에 텍스처 이미지를 전송의 소스 및 대상으로 사용하려는 의도를 알려야 합니다. `createTextureImage`에서 텍스처 이미지의 사용 플래그에 `VK_IMAGE_USAGE_TRANSFER_SRC_BIT`를 추가하세요:

```c++
...
createImage(texWidth, texHeight, mipLevels, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_TRANSFER_SRC_BIT | VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, textureImage, textureImageMemory);
...
```

다른 이미지 작업과 마찬가지로, `vkCmdBlitImage`는 작업하는 이미지의 레이아웃에 따라 달라집니다. 전체 이미지를 `VK_IMAGE_LAYOUT_GENERAL`로 전환할 수 있지만, 이는 아마도 느릴 것입니다. 최적의 성능을 위해 소스 이미지는 `VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL`이어야 하며, 대상 이미지는 `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`이어야 합니다. Vulkan은 이미지의 각 밉맵 레벨을 독립적으로 전환할 수 있습니다. 각 blit는 한 번에 두 개의 밉맵 레벨만 다루므로, blit 명령 사이에 각 레벨을 최적의 레이아웃으로 전환할 수 있습니다.

`transitionImageLayout`은 전체 이미지에 대해서만 레이아웃 전환을 수행하므로, 몇 가지 추가 파이프라인 배리어 명령을 작성해야 합니다. `createTextureImage`에서 `VK_IMAGE_LAYOUT_SHADER

_READ_ONLY_OPTIMAL`로의 기존 전환을 제거하세요:

```c++
...
transitionImageLayout(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, mipLevels);
    copyBufferToImage(stagingBuffer, textureImage, static_cast<uint32_t>(texWidth), static_cast<uint32_t>(texHeight));
//transitioned to VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL while generating mipmaps
...
```

이렇게 하면 텍스처 이미지의 각 레벨이 `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`에 남게 됩니다. 각 레벨은 읽기 작업이 끝난 후 `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL`로 전환됩니다.

이제 밉맵을 생성하는 함수를 작성할 것입니다:

```c++
void generateMipmaps(VkImage image, int32_t texWidth, int32_t texHeight, uint32_t mipLevels) {
    VkCommandBuffer commandBuffer = beginSingleTimeCommands();

    VkImageMemoryBarrier barrier{};
    barrier.sType = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER;
    barrier.image = image;
    barrier.srcQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
    barrier.dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
    barrier.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
    barrier.subresourceRange.baseArrayLayer = 0;
    barrier.subresourceRange.layerCount = 1;
    barrier.subresourceRange.levelCount = 1;

    endSingleTimeCommands(commandBuffer);
}
```

여러 전환을 수행할 것이므로 이 `VkImageMemoryBarrier`를 재사용할 것입니다. 위에서 설정된 필드는 모든 배리어에 대해 동일하게 유지됩니다. `subresourceRange.miplevel`, `oldLayout`, `newLayout`, `srcAccessMask`, `dstAccessMask`는 각 전환마다 변경됩니다.

```c++
int32_t mipWidth = texWidth;
int32_t mipHeight = texHeight;

for (uint32_t i = 1; i < mipLevels; i++) {

}
```

이 루프는 각 `VkCmdBlitImage` 명령을 기록할 것입니다. 루프 변수가 0이 아니라 1에서 시작한다는 점에 유의하세요.

```c++
barrier.subresourceRange.baseMipLevel = i - 1;
barrier.oldLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL;
barrier.newLayout = VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL;
barrier.srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
barrier.dstAccessMask = VK_ACCESS_TRANSFER_READ_BIT;

vkCmdPipelineBarrier(commandBuffer,
    VK_PIPELINE_STAGE_TRANSFER_BIT, VK_PIPELINE_STAGE_TRANSFER_BIT, 0,
    0, nullptr,
    0, nullptr,
    1, &barrier);
```

먼저 레벨 `i - 1`을 `VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL`로 전환합니다. 이 전환은 레벨 `i - 1`이 채워지기를 기다립니다. 이는 이전 blit 명령 또는 `vkCmdCopyBufferToImage`에서 가져온 것일 수 있습니다. 현재 blit 명령은 이 전환을 기다릴 것입니다.

```c++
VkImageBlit blit{};
blit.srcOffsets[0] = { 0, 0, 0 };
blit.srcOffsets[1] = { mipWidth, mipHeight, 1 };
blit.srcSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
blit.srcSubresource.mipLevel = i - 1;
blit.srcSubresource.baseArrayLayer = 0;
blit.srcSubresource.layerCount = 1;
blit.dstOffsets[0] = { 0, 0, 0 };
blit.dstOffsets[1] = { mipWidth > 1 ? mipWidth / 2 : 1, mipHeight > 1 ? mipHeight / 2 : 1, 1 };
blit.dstSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
blit.dstSubresource.mipLevel = i;
blit.dstSubresource.baseArrayLayer = 0;
blit.dstSubresource.layerCount = 1;
```

다음으로, blit 작업에서 사용될 영역을 지정합니다. 소스 밉맵 레벨은 `i - 1`이고 대상 밉맵 레벨은 `i`입니다. `srcOffsets` 배열의 두 요소는 데이터가 blit될 3D 영역을 결정합니다. `dstOffsets`는 데이터가 blit될 영역을 결정합니다. `dstOffsets[1]`의 X 및 Y 치수는 이전 레벨의 절반 크기이므로 2로 나눕니다. `srcOffsets[1]` 및 `dstOffsets[1]`의 Z 치수는 1이어야 합니다. 왜냐하면 2D 이미지는 깊이가 1이기 때문입니다.

```c++
vkCmdBlitImage(commandBuffer,
    image, VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL,
    image, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL,
    1, &blit,
    VK_FILTER_LINEAR);
```

이제 blit 명령을 기록합니다. `textureImage`가 `srcImage` 및 `dstImage` 매개변수 모두에 사용됩니다. 이는 동일한 이미지의 다른 레벨 간에 blit되기 때문입니다. 소스 밉맵 레벨은 방금 `VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL`로 전환되었고 대상 레벨은 `createTextureImage`에서 `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`에 있습니다.

전용 전송 큐를 사용하는 경우(`Vertex buffers`에서 제안한 바와 같이) 주의하십시오: `vkCmdBlitImage`는 그래픽 기능이 있는 큐에 제출되어야 합니다.

마지막 매개변수를 사용하여 blit에서 사용할 `VkFilter`를 지정할 수 있습니다. `VkSampler`를 만들 때 가진 필터링 옵션과 동일한 옵션이 여기에 있습니다. 보간을 활성화하려면 `VK_FILTER_LINEAR`를 사용합니다.

```c++
barrier.oldLayout = VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL;
barrier.newLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
barrier.srcAccessMask = VK_ACCESS_TRANSFER_READ_BIT;
barrier.dstAccessMask = VK_ACCESS_SHADER_READ_BIT;

vkCmdPipelineBarrier(commandBuffer,
    VK_PIPELINE_STAGE_TRANSFER_BIT, VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT, 0,
    0, nullptr,
    0, nullptr,
    1, &barrier);
```

이 배리어는 밉맵 레벨 `i - 1`을 `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL`로 전환합니다. 이 전환은 현재 blit 명령이 끝나기를 기다립니다. 모든 샘플링 작업은 이 전환을 끝나기를 기다릴 것입니다.

```c++
    ...
    if (mipWidth > 1) mipWidth /= 2;
    if (mipHeight > 1) mipHeight /= 2;
```

루프의 끝에서 현재 밉맵 치수를 2로 나눕니다. 이는 각 치수를 나누기 전에 확인하여 그 치수가 결코 0이 되지 않도록 합니다. 이는 이미지가 정사각형이 아닌 경우를 처리합니다. 왜냐하면 밉맵 치수 중 하나가 다른 치수보다 먼저 1에 도달할 것이기 때문입니다. 이 경우 해당 치수는 남은 모든 레벨에 대해 1이어야 합니다.

```c++
    barrier.subresourceRange.baseMipLevel = mipLevels - 1;
    barrier.oldLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL;
    barrier.newLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
    barrier.srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
    barrier.dstAccessMask = VK_ACCESS_SHADER_READ_BIT;

    vkCmdPipelineBarrier(commandBuffer,
        VK_PIPELINE_STAGE_TRANSFER_BIT, VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT, 0,
        0, nullptr,
        0, nullptr,
        1, &barrier);

    endSingleTimeCommands(commandBuffer);
}
```

명령 버퍼를 종료하기 전에 하나 더 파이

프라인 배리어를 삽입합니다. 이 배리어는 마지막 밉맵 레벨을 `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`에서 `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL`로 전환합니다. 이는 루프에서 처리되지 않았습니다. 왜냐하면 마지막 밉맵 레벨은 결코 blit되지 않기 때문입니다.

마지막으로 `createTextureImage`에서 `generateMipmaps` 호출을 추가하세요:

```c++
transitionImageLayout(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, mipLevels);
    copyBufferToImage(stagingBuffer, textureImage, static_cast<uint32_t>(texWidth), static_cast<uint32_t>(texHeight));
//transitioned to VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL while generating mipmaps
...
generateMipmaps(textureImage, texWidth, texHeight, mipLevels);
```

이제 텍스처 이미지의 밉맵이 완전히 채워졌습니다.

## 선형 필터링 지원

`vkCmdBlitImage`와 같은 내장 함수를 사용하여 모든 밉맵 레벨을 생성하는 것은 매우 편리하지만, 불행히도 모든 플랫폼에서 지원되는 것은 보장되지 않습니다. 사용하는 텍스처 이미지 형식이 선형 필터링을 지원해야 하는데, 이는 `vkGetPhysicalDeviceFormatProperties` 함수로 확인할 수 있습니다. `generateMipmaps` 함수에 이를 확인하는 절차를 추가할 것입니다.

먼저 이미지 형식을 지정하는 추가 매개변수를 추가하세요:

```c++
void createTextureImage() {
    ...

    generateMipmaps(textureImage, VK_FORMAT_R8G8B8A8_SRGB, texWidth, texHeight, mipLevels);
}

void generateMipmaps(VkImage image, VkFormat imageFormat, int32_t texWidth, int32_t texHeight, uint32_t mipLevels) {

    ...
}
```

`generateMipmaps` 함수에서 `vkGetPhysicalDeviceFormatProperties`를 사용하여 텍스처 이미지 형식의 속성을 요청하세요:

```c++
void generateMipmaps(VkImage image, VkFormat imageFormat, int32_t texWidth, int32_t texHeight, uint32_t mipLevels) {

    // 선형 블리팅이 이미지 형식을 지원하는지 확인
    VkFormatProperties formatProperties;
    vkGetPhysicalDeviceFormatProperties(physicalDevice, imageFormat, &formatProperties);

    ...
```

`VkFormatProperties` 구조체에는 사용 방식에 따라 형식을 사용할 수 있는 방법을 설명하는 `linearTilingFeatures`, `optimalTilingFeatures`, `bufferFeatures`라는 세 개의 필드가 있습니다. 우리는 최적 타일링 형식으로 텍스처 이미지를 생성하므로 `optimalTilingFeatures`를 확인해야 합니다. 선형 필터링 기능은 `VK_FORMAT_FEATURE_SAMPLED_IMAGE_FILTER_LINEAR_BIT`로 확인할 수 있습니다:

```c++
if (!(formatProperties.optimalTilingFeatures & VK_FORMAT_FEATURE_SAMPLED_IMAGE_FILTER_LINEAR_BIT)) {
    throw std::runtime_error("texture image format does not support linear blitting!");
}
```

이 경우에는 두 가지 대안이 있습니다. 선형 블리팅을 지원하는 *다른* 일반 텍스처 이미지 형식을 찾는 함수를 구현하거나, [stb_image_resize](https://github.com/nothings/stb/blob/master/stb_image_resize.h)와 같은 라이브러리로 소프트웨어에서 밉맵 생성을 구현할 수 있습니다. 각 밉맵 레벨은 원본 이미지를 로드한 것과 동일한 방식으로 이미지에 로드될 수 있습니다.

일반적으로 런타임에 밉맵 레벨을 생성하는 것은 드뭅니다. 일반적으로 로딩 속도를 향상시키기 위해 기본 레벨과 함께 텍스처 파일에 미리 생성되어 저장됩니다. 소프트웨어에서 크기를 조절하고 파일에서 여러 레벨을 로드하는 것은 독자에게 남겨진 연습입니다.

## 샘플러

`VkImage`는 밉맵 데이터를 보유하지만 `VkSampler`는 렌더링하는 동안 해당 데이터가 어떻게 읽히는지를 제어합니다. Vulkan은 `minLod`, `maxLod`, `mipLodBias`, `mipmapMode`("Lod"는 "세부 수준"을 의미)를 지정할 수 있게 해줍니다. 텍스처가 샘플링될 때, 샘플러는 다음 의사 코드에 따라 밉맵 레벨을 선택합니다:

```c++
lod = getLodLevelFromScreenSize(); //객체가 가까울 때 작을 수 있으며 음수일 수 있음
lod = clamp(lod + mipLodBias, minLod, maxLod);

level = clamp(floor(lod), 0, texture.mipLevels - 1);  //텍스처의 밉맵 레벨 수에 맞춰 클램핑됨

if (mipmapMode == VK_SAMPLER_MIPMAP_MODE_NEAREST) {
    color = sample(level);
} else {
    color = blend(sample(level), sample(level + 1));
}
```

`samplerInfo.mipmapMode`가 `VK_SAMPLER_MIPMAP_MODE_NEAREST`인 경우, `lod`는 샘플링할 밉맵 레벨을 선택합니다. mipmap 모드가 `VK_SAMPLER_MIPMAP_MODE_LINEAR`인 경우, `lod`는 샘플링할 두 밉맵 레벨을 선택하는 데 사용됩니다. 이들 레벨은 샘플링되고 결과는 선형적으로 혼합됩니다.

샘플 작업도 `lod`의 영향을 받습니다:

```c++
if (lod <= 0) {
    color = readTexture(uv, magFilter);
} else {
    color = readTexture(uv, minFilter);
}
```

객체가 카메라에 가까우면 `magFilter`가 필터로 사용됩니다. 객체가 카메라에서 멀어지면 `minFilter`가 사용됩니다. 일반적으로 `lod`는 음수가 아니며, 카메라에 가까울 때만 0입니다. `mipLodBias`를 사용하면 일반적으로 사용할 것보다 낮은 `lod`와 `level`을 강제로 사용할 수 있습니다.

이 장의 결과를 보려면 `textureSampler`에 대한 값을 선택해야 합니다. 이미 `minFilter`와 `magFilter`를 `VK_FILTER_LINEAR`를 사용하도록 설정했습니다. `minLod`, `maxLod`, `mipLodBias`, `mipmapMode`에 대한 값을 선택하기만 하면 됩니다.

```c++
void createTextureSampler() {
    ...
    samplerInfo.mipmapMode = VK_SAMPLER_MIPMAP_MODE_LINEAR;
    samplerInfo.minLod = 0.0f; // 선택 사항
    samplerInfo.maxLod = VK_LOD_CLAMP_NONE;
    samplerInfo.mipLodBias = 0.0f; // 선택 사항
    ...
}
```

모든 밉맵 레벨을 사용할 수 있도록 허용하려면 `minLod`를 0.0f로 설정하고 `maxLod`를 `VK_LOD_CLAMP_NONE`으로 설정합니다. 이 상수는 1000.0f와 같으며, 이는 텍스처에서 사용 가능한 모든 밉맵 레벨이 샘플링될 것임을 의미합니다.

 `lod` 값을 변경할 이유가 없으므로 `mipLodBias`를 0.0f로 설정합니다.

이제 프로그램을 실행하면 다음과 같은 결과를 볼 수 있습니다:

![](/images/mipmaps.png)

우리의 장면이 매우 간단하기 때문에 큰 차이는 아니지만, 자세히 보면 미묘한 차이가 있습니다.

![](/images/mipmaps_comparison.png)

가장 눈에 띄는 차이는 종이에 쓰여진 글입니다. 밉맵을 사용하면 글이 부드럽게 처리됩니다. 밉맵을 사용하지 않으면 글에 거친 가장자리와 모아레 아티팩트로 인한 간격이 생깁니다.

샘플러 설정을 변경하여 밉맵이 어떻게 영향을 받는지 실험해 볼 수 있습니다. 예를 들어, `minLod`를 변경하여 샘플러가 가장 낮은 밉맵 레벨을 사용하지 않도록 강제할 수 있습니다:

```c++
samplerInfo.minLod = static_cast<float>(mipLevels / 2);
```

이 설정은 다음 이미지를 생성합니다:

![](/images/highmipmaps.png)

이것은 객체가 카메라에서 멀어질 때 더 높은 밉맵 레벨이 사용되는 방식입니다.

[C++ 코드](/code/29_mipmapping.cpp) /
[버텍스 셰이더](/code/27_shader_depth.vert) /
[프래그먼트 셰이더](/code/27_shader_depth.frag)