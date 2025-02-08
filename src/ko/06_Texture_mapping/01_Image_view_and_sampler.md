# 이미지 뷰 및 샘플러

## 소개

이번 장에서는 이미지를 샘플링하는 데 필요한 두 가지 리소스를 생성합니다. 첫 번째 리소스는 스왑 체인 이미지를 다루면서 이미 본 적이 있는 것이지만, 두 번째 리소스는 새로운 것으로, 셰이더가 이미지에서 텍셀을 읽는 방식과 관련이 있습니다.

## 텍스처 이미지 뷰

스왑 체인 이미지와 프레임버퍼에서 보았듯이, 이미지는 직접 접근하는 대신 이미지 뷰를 통해 접근됩니다. 텍스처 이미지에 대해서도 이미지 뷰를 생성해야 합니다.

텍스처 이미지에 대한 `VkImageView`를 보관할 클래스 멤버를 추가하고 `createTextureImageView`라는 새 함수를 만들어 그곳에서 이미지 뷰를 생성합니다:

```c++
VkImageView textureImageView;

...

void initVulkan() {
    ...
    createTextureImage();
    createTextureImageView();
    createVertexBuffer();
    ...
}

...

void createTextureImageView() {

}
```

이 함수는 `createImageViews`에서 직접 코드를 기반으로 할 수 있습니다. 변경해야 할 것은 `format`과 `image` 두 가지뿐입니다:

```c++
VkImageViewCreateInfo viewInfo{};
viewInfo.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
viewInfo.image = textureImage;
viewInfo.viewType = VK_IMAGE_VIEW_TYPE_2D;
viewInfo.format = VK_FORMAT_R8G8B8A8_SRGB;
viewInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
viewInfo.subresourceRange.baseMipLevel = 0;
viewInfo.subresourceRange.levelCount = 1;
viewInfo.subresourceRange.baseArrayLayer = 0;
viewInfo.subresourceRange.layerCount = 1;
```

`viewInfo.components` 초기화를 명시적으로 생략했습니다. 왜냐하면 `VK_COMPONENT_SWIZZLE_IDENTITY`가 어차피 `0`으로 정의되어 있기 때문입니다. `vkCreateImageView`를 호출하여 이미지 뷰를 생성을 완료하세요:

```c++
if (vkCreateImageView(device, &viewInfo, nullptr, &textureImageView) != VK_SUCCESS) {
    throw std::runtime_error("failed to create texture image view!");
}
```

로직이 `createImageViews`에서 많이 중복되므로, 새로운 `createImageView` 함수로 추상화하는 것이 좋습니다:

```c++
VkImageView createImageView(VkImage image, VkFormat format) {
    VkImageViewCreateInfo viewInfo{};
    viewInfo.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
    viewInfo.image = image;
    viewInfo.viewType = VK_IMAGE_VIEW_TYPE_2D;
    viewInfo.format = format;
    viewInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
    viewInfo.subresourceRange.baseMipLevel = 0;
    viewInfo.subresourceRange.levelCount = 1;
    viewInfo.subresourceRange.baseArrayLayer = 0;
    viewInfo.subresourceRange.layerCount = 1;

    VkImageView imageView;
    if (vkCreateImageView(device, &viewInfo, nullptr, &imageView) != VK_SUCCESS) {
        throw std::runtime_error("failed to create image view!");
    }

    return imageView;
}
```

이제 `createTextureImageView` 함수는 다음과 같이 간소화할 수 있습니다:

```c++
void createTextureImageView() {
    textureImageView = createImageView(textureImage, VK_FORMAT_R8G8B8A8_SRGB);
}
```

그리고 `createImageViews`도 간소화할 수 있습니다:

```c++
void createImageViews() {
    swapChainImageViews.resize(swapChainImages.size());

    for (uint32_t i = 0; i < swapChainImages.size(); i++) {
        swapChainImageViews[i] = createImageView(swapChainImages[i], swapChainImageFormat);
    }
}
```

프로그램의 끝에서 이미지 자체를 파괴하기 직전에 이미지 뷰를 파괴하세요:

```c++
void cleanup() {
    cleanupSwapChain();

    vkDestroyImageView(device, textureImageView, nullptr);

    vkDestroyImage(device, textureImage, nullptr);
    vkFreeMemory(device, textureImageMemory, nullptr);
```

## 샘플러

셰이더에서 직접 이미지에서 텍셀을 읽을 수 있지만, 일반적으로 텍스처로 사용될 때는 그

렇게 하지 않습니다. 텍스처는 보통 샘플러를 통해 접근되며, 샘플러는 최종 색상을 검색하는 데 필요한 필터링 및 변환을 적용합니다.

이 필터는 오버샘플링과 같은 문제를 다루는 데 도움이 됩니다. 예를 들어, 텍셀보다 많은 프래그먼트에 매핑된 텍스처를 고려해 보세요. 각 프래그먼트의 텍스처 좌표에 가장 가까운 텍셀을 단순히 가져오면 첫 번째 이미지와 같은 결과를 얻게 됩니다:

![](/images/texture_filtering.png)

4개의 가장 가까운 텍셀을 선형 보간을 통해 결합하면 오른쪽과 같은 더 부드러운 결과를 얻을 수 있습니다. 물론 귀하의 애플리케이션에는 왼쪽 스타일이 더 적합한 예술 스타일 요구 사항이 있을 수 있습니다(마인크래프트를 생각해 보세요), 하지만 일반적인 그래픽 애플리케이션에서는 오른쪽이 선호됩니다. 샘플러 객체는 텍스처에서 색상을 읽을 때 자동으로 이 필터링을 적용합니다.

언더샘플링은 반대 문제로, 텍셀보다 프래그먼트가 더 많습니다. 이는 예를 들어 날카로운 각도에서 체크보드 텍스처를 샘플링할 때 아티팩트를 유발합니다:

![](/images/anisotropic_filtering.png)

왼쪽 이미지에서 보듯이, 멀리서 보면 텍스처가 흐릿하게 보입니다. 이 문제의 해결책은 [등방성 필터링](https://en.wikipedia.org/wiki/Anisotropic_filtering)이며, 이 또한 샘플러에 의해 자동으로 적용될 수 있습니다.

이 필터 외에도 샘플러는 변환을 처리할 수 있습니다. *주소 지정 모드*를 통해 이미지 밖의 텍셀을 읽으려고 할 때 발생하는 일을 결정합니다. 아래 이미지는 가능한 몇 가지 옵션을 보여줍니다:

![](/images/texture_addressing.png)

이제 `createTextureSampler`라는 함수를 만들어 이러한 샘플러 객체를 설정할 것입니다. 나중에 이 샘플러를 사용하여 셰이더에서 텍스처로부터 색상을 읽을 것입니다.

```c++
void initVulkan() {
    ...
    createTextureImage();
    createTextureImageView();
    createTextureSampler();
    ...
}

...

void createTextureSampler() {

}
```

샘플러는 `VkSamplerCreateInfo` 구조체를 통해 설정되며, 적용할 모든 필터와 변환을 지정합니다.

```c++
VkSamplerCreateInfo samplerInfo{};
samplerInfo.sType = VK_STRUCTURE_TYPE_SAMPLER_CREATE_INFO;
samplerInfo.magFilter = VK_FILTER_LINEAR;
samplerInfo.minFilter = VK_FILTER_LINEAR;
```

`magFilter` 및 `minFilter` 필드는 확대 또는 축소된 텍셀을 보간하는 방법을 지정합니다. 확대는 위에서 설명한 오버샘플링 문제와 관련이 있으며, 축소는 언더샘플링과 관련이 있습니다. 선택할 수 있는 옵션은 `VK_FILTER_NEAREST` 및 `VK_FILTER_LINEAR`로, 위 이미지에서 보여진 모드와 대응합니다.

```c++
samplerInfo.addressModeU = VK_SAMPLER_ADDRESS_MODE_REPEAT;
samplerInfo.addressModeV = VK_SAMPLER_ADDRESS_MODE_REPEAT;
samplerInfo.addressModeW = VK_SAMPLER_ADDRESS_MODE_REPEAT;
```

`addressMode` 필드를 사용하여 축별로 주소 지정 모드를 지정할 수 있습니다. 사용 가능한 값은 아래에 나열되어 있습니다. 대부분은 위의 이미지에서 보여진 것과 같습니다. 축은 X, Y, Z가 아닌 U, V, W로 불리는 것이 텍스처 공간 좌표에 대한 관례입니다.

- `VK_SAMPLER_ADDRESS_MODE_REPEAT`: 이미지 차원을 넘어갈 때 텍스처를 반복합니다.
- `VK_SAMPLER_ADDRESS_MODE_MIRRORED_REPEAT`: 반복과 비슷하지만 차원을 넘어갈 때 좌표를 반전시켜 이미지를 거울처럼 보입니다.
- `VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE`: 이미지 차원을 넘어갈 때 가장 가까운 가장자리의 색상을 취합니다.
- `VK_SAMPLER_ADDRESS_MODE_MIRROR_CLAMP_TO_EDGE`: 가장자리에 고정하는 것과 비슷하지만, 가장 가까운 가장자리가 아닌 반대 가장자리를 사용합니다.
- `VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_BORDER`: 이미지 차원을 넘어갈 때 고체 색상을 반환합니다.

여기서 어떤 주소 지정 모드를 사용하든 중요하지 않습니다. 왜냐하면 이 튜토리얼에서는 이미지 밖을 샘플링하지 않기 때문입니다. 그러나 반복 모드는 바닥과 벽과 같은 텍스처를 타일링하는 데 사용될 수 있으므로 가장 일반적인 모드일 수 있습니다.

```c++
samplerInfo.anisotropyEnable = VK_TRUE;
samplerInfo.maxAnisotropy = ???;
```

이 두 필드는 등방성 필터링을 사용할지 여부를 지정합니다. 성능이 문제가 되지 않는 한 사용하는 것이 좋습니다. `maxAnisotropy` 필드는 최종 색상을 계산하는 데 사용될 수 있는 텍셀 샘플 수를 제한합니다. 값이 낮을수록 성능은 좋아지지만 결과 품질은 떨어집니다. 사용할 수 있는 값을 결정하려면 물리적 장치의 속성을 검색해야 합니다.

```c++
VkPhysicalDeviceProperties properties{};
vkGetPhysicalDeviceProperties(physicalDevice, &properties);
```

`VkPhysicalDeviceProperties` 구조체의 문서를 살펴보면 `limits`라는 이름의 `VkPhysicalDeviceLimits` 멤버를 포함한다는 것을 알 수 있습니다. 이 구조체는 다시 `maxSamplerAnisotropy`라는 멤버를 가지고 있으며, 이는 `maxAnisotropy`에 지정할 수 있는 최대값입니다. 최대 품질을 원한다면 이 값을 직접 사용할 수 있습니다:

```c++
samplerInfo.maxAnisotropy = properties.limits.maxSamplerAnisotropy;
```

프로그램의 시작 부분에서 이 속성을 쿼리하고 필요한 함수로 전달하거나 `createTextureSampler` 함수 자체에서 쿼리할 수 있습니다.

```c++
samplerInfo.borderColor = VK_BORDER_COLOR_INT_OPAQUE_BLACK;
```

`borderColor` 필드는 클램프 투 보더 주소 지정 모드로 이미지 차원을 넘어 샘플링할 때 반환되는 색상을 지정합니다. 가능한 값은 검은색, 흰색 또는 투명한 색상이며, float 또

는 int 형식일 수 있습니다. 임의의 색상을 지정할 수는 없습니다.

```c++
samplerInfo.unnormalizedCoordinates = VK_FALSE;
```

`unnormalizedCoordinates` 필드는 텍셀을 이미지에서 주소 지정하는 데 사용하려는 좌표계를 지정합니다. 이 필드가 `VK_TRUE`이면 `[0, texWidth)` 및 `[0, texHeight)` 범위 내의 좌표를 간단히 사용할 수 있습니다. `VK_FALSE`인 경우 모든 축에서 `[0, 1)` 범위를 사용하여 텍셀을 주소 지정합니다. 실제 애플리케이션에서는 거의 항상 정규화된 좌표를 사용합니다. 그렇게 하면 정확히 같은 좌표를 사용하여 다양한 해상도의 텍스처를 사용할 수 있습니다.

```c++
samplerInfo.compareEnable = VK_FALSE;
samplerInfo.compareOp = VK_COMPARE_OP_ALWAYS;
```

비교 함수가 활성화되면 텍셀은 먼저 값과 비교되고, 그 비교 결과는 필터링 작업에 사용됩니다. 이는 주로 [퍼센트 클로저 필터링](https://developer.nvidia.com/gpugems/GPUGems/gpugems_ch11.html)에 사용되며, 그림자 맵에서 사용됩니다. 이에 대해서는 미래의 장에서 살펴볼 것입니다.

```c++
samplerInfo.mipmapMode = VK_SAMPLER_MIPMAP_MODE_LINEAR;
samplerInfo.mipLodBias = 0.0f;
samplerInfo.minLod = 0.0f;
samplerInfo.maxLod = 0.0f;
```

이 모든 필드는 미핑에 적용됩니다. [나중에 나오는 장](/Generating_Mipmaps)에서 미핑을 살펴볼 것이지만, 기본적으로 적용될 수 있는 또 다른 유형의 필터입니다.

이제 샘플러의 기능이 완전히 정의되었습니다. 샘플러 객체의 핸들을 보관할 클래스 멤버를 추가하고 `vkCreateSampler`를 사용하여 샘플러를 생성하세요:

```c++
VkImageView textureImageView;
VkSampler textureSampler;

...

void createTextureSampler() {
    ...

    if (vkCreateSampler(device, &samplerInfo, nullptr, &textureSampler) != VK_SUCCESS) {
        throw std::runtime_error("failed to create texture sampler!");
    }
}
```

샘플러는 어디에도 `VkImage`를 참조하지 않습니다. 샘플러는 텍스처에서 색상을 추출하는 인터페이스를 제공하는 독립적인 객체입니다. 원하는 이미지에 적용할 수 있으며, 1D, 2D 또는 3D일 수 있습니다. 이는 많은 오래된 API와 다르며, 이러한 API는 텍스처 이미지와 필터링을 단일 상태로 결합했습니다.

프로그램 끝에서 이미지에 더 이상 액세스하지 않을 때 샘플러를 파괴하세요:

```c++
void cleanup() {
    cleanupSwapChain();

    vkDestroySampler(device, textureSampler, nullptr);
    vkDestroyImageView(device, textureImageView, nullptr);

    ...
}
```

## 등방성 디바이스 기능

지금 프로그램을 실행하면 다음과 같은 유효성 검사 계층 메시지를 볼 수 있습니다:

![](/images/validation_layer_anisotropy.png)

그 이유는 등방성 필터링이 실제로 선택적 디바이스 기능이기 때문입니다. `createLogicalDevice` 함수를 업데이트하여 이를 요청해야 합니다:

```c++
VkPhysicalDeviceFeatures deviceFeatures{};
deviceFeatures.samplerAnisotropy = VK_TRUE;
```

현대 그래픽 카드가 이를 지원하지 않을 가능성은 매우 낮지만, `isDeviceSuitable`을 업데이트하여 이를 사용할 수 있는지 확인해야 합니다:

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    ...

    VkPhysicalDeviceFeatures supportedFeatures;
    vkGetPhysicalDeviceFeatures(device, &supportedFeatures);

    return indices.isComplete() && extensionsSupported && swapChainAdequate && supportedFeatures.samplerAnisotropy;
}
```

`vkGetPhysicalDeviceFeatures`는 지원되는 기능을 나타내기 위해

 `VkPhysicalDeviceFeatures` 구조체를 재사용합니다. 불리언 값으로 설정됩니다.

등방성 필터링을 강제로 사용할 필요는 없으며, 다음과 같이 조건부로 설정할 수 있습니다:

```c++
samplerInfo.anisotropyEnable = VK_FALSE;
samplerInfo.maxAnisotropy = 1.0f;
```

다음 장에서는 이미지와 샘플러 객체를 셰이더에 노출하여 사각형에 텍스처를 그릴 것입니다.

[C++ 코드](/code/25_sampler.cpp) /
[버텍스 셰이더](/code/22_shader_ubo.vert) /
[프래그먼트 셰이더](/code/22_shader_ubo.frag)