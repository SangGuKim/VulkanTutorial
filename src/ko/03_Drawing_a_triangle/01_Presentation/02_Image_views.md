# 이미지 뷰

스왑 체인의 이미지를 포함한 모든 `VkImage`를 렌더 파이프라인에서 사용하기 위해서는 `VkImageView` 객체를 생성해야 합니다. 이미지 뷰는 말 그대로 이미지를 바라보는 뷰입니다. 이미지를 어떻게 접근할지, 그리고 이미지의 어느 부분에 접근할지를 설명합니다. 예를 들어, 밉맵 레벨 없이 2D 텍스처 깊이 텍스처로 취급해야 하는지 등을 설정할 수 있습니다.

이번 장에서는 `createImageViews` 함수를 작성하여 스왑 체인의 모든 이미지에 대해 기본 이미지 뷰를 생성할 것입니다. 이렇게 생성된 이미지 뷰는 나중에 컬러 타겟으로 사용될 것입니다.

먼저 이미지 뷰를 저장할 클래스 멤버를 추가합니다:

```c++
std::vector<VkImageView> swapChainImageViews;
```

`createImageViews` 함수를 생성하고 스왑 체인 생성 직후에 호출하도록 합니다.

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
    createImageViews();
}

void createImageViews() {

}
```

가장 먼저 해야 할 일은 생성할 모든 이미지 뷰를 담을 수 있도록 리스트의 크기를 조정하는 것입니다:

```c++
void createImageViews() {
    swapChainImageViews.resize(swapChainImages.size());

}
```

다음으로, 모든 스왑 체인 이미지를 순회하는 루프를 설정합니다.

```c++
for (size_t i = 0; i < swapChainImages.size(); i++) {

}
```

이미지 뷰 생성을 위한 매개변수는 `VkImageViewCreateInfo` 구조체에 지정됩니다. 처음 몇 개의 매개변수는 간단합니다.

```c++
VkImageViewCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
createInfo.image = swapChainImages[i];
```

`viewType`과 `format` 필드는 이미지 데이터를 어떻게 해석할지 지정합니다. `viewType` 매개변수를 통해 이미지를 1D 텍스처, 2D 텍스처, 3D 텍스처 및 큐브맵으로 취급할 수 있습니다.

```c++
createInfo.viewType = VK_IMAGE_VIEW_TYPE_2D;
createInfo.format = swapChainImageFormat;
```

`components` 필드를 사용하면 색상 채널을 서로 바꿀 수 있습니다. 예를 들어, 모노크롬 텍스처를 위해 모든 채널을 빨간색 채널에 매핑할 수 있습니다. 채널에 `0`과 `1`의 상수값을 매핑할 수도 있습니다. 우리의 경우에는 기본 매핑을 사용하겠습니다.

```c++
createInfo.components.r = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.g = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.b = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.a = VK_COMPONENT_SWIZZLE_IDENTITY;
```

`subresourceRange` 필드는 이미지의 용도와 이미지의 어느 부분에 접근해야 하는지를 설명합니다. 우리의 이미지는 밉맵 레벨이나 다중 레이어 없이 컬러 타겟으로 사용될 것입니다.

```c++
createInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
createInfo.subresourceRange.baseMipLevel = 0;
createInfo.subresourceRange.levelCount = 1;
createInfo.subresourceRange.baseArrayLayer = 0;
createInfo.subresourceRange.layerCount = 1;
```

만약 입체 3D 애플리케이션을 작업하고 있다면, 여러 레이어가 있는 스왑 체인을 생성할 수 있습니다. 그런 다음 서로 다른 레이어에 접근하여 왼쪽 눈과 오른쪽 눈의 뷰를 나타내는 각 이미지에 대해 여러 이미지 뷰를 생성할 수 있습니다.

이제 `vkCreateImageView`를 호출하여 이미지 뷰를 생성할 수 있습니다:

```c++
if (vkCreateImageView(device, &createInfo, nullptr, &swapChainImageViews[i]) != VK_SUCCESS) {
    throw std::runtime_error("failed to create image views!");
}
```

이미지와 달리 이미지 뷰는 우리가 명시적으로 생성했기 때문에, 프로그램 종료 시 이를 제거하기 위한 유사한 루프를 추가해야 합니다:

```c++
void cleanup() {
    for (auto imageView : swapChainImageViews) {
        vkDestroyImageView(device, imageView, nullptr);
    }

    ...
}
```

이미지 뷰는 이미지를 텍스처로 사용하기에 충분하지만, 아직 렌더 타겟으로 사용하기에는 부족합니다. 이를 위해서는 프레임버퍼라고 하는 한 단계의 간접 참조가 더 필요합니다. 하지만 먼저 그래픽스 파이프라인을 설정해야 합니다.

[C++ 코드](/code/07_image_views.cpp)