# 렌더 패스 (Render passes)

## 설정

파이프라인 생성을 마무리하기 전에, 렌더링하는 동안 사용될 프레임버퍼 어태치먼트(attachment, 첨부되는—렌더링 결과가 저장될—버퍼)들에 대해 Vulkan에게 알려주어야 합니다. 우리는 컬러 버퍼와 깊이 버퍼가 각각 몇 개가 있을지, 각각에 대해 몇 개의 샘플을 사용할지, 그리고 렌더링 작업 전반에 걸쳐 이들의 내용을 어떻게 처리할지 지정해야 합니다. 이 모든 정보는 *렌더 패스* 객체에 포함되며, 이를 위해 새로운 `createRenderPass` 함수를 만들 것입니다. 이 함수를 `initVulkan`에서 `createGraphicsPipeline` 전에 호출하세요.

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
    createImageViews();
    createRenderPass();
    createGraphicsPipeline();
}

...

void createRenderPass() {

}
```

## 어태치먼트 설명

우리의 경우에는 스왑 체인의 이미지 중 하나로 표현되는 단일 컬러 버퍼 어태치먼트만 가지게 될 것입니다.

```c++
void createRenderPass() {
    VkAttachmentDescription colorAttachment{};
    colorAttachment.format = swapChainImageFormat;
    colorAttachment.samples = VK_SAMPLE_COUNT_1_BIT;
}
```

컬러 어태치먼트의 `format`은 스왑 체인 이미지의 형식과 일치해야 하며, 아직 멀티샘플링을 사용하지 않으므로 1개의 샘플을 사용하겠습니다.

```c++
colorAttachment.loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
colorAttachment.storeOp = VK_ATTACHMENT_STORE_OP_STORE;
```

`loadOp`와 `storeOp`는 렌더링 전과 후에 어태치먼트의 데이터를 어떻게 처리할지 결정합니다. `loadOp`에 대해서는 다음과 같은 선택지가 있습니다:

* `VK_ATTACHMENT_LOAD_OP_LOAD`: 어태치먼트의 기존 내용을 보존
* `VK_ATTACHMENT_LOAD_OP_CLEAR`: 시작 시 값을 상수로 초기화
* `VK_ATTACHMENT_LOAD_OP_DONT_CARE`: 기존 내용이 정의되지 않음; 신경 쓰지 않음

우리의 경우에는 새 프레임을 그리기 전에 프레임버퍼를 검은색으로 초기화하기 위해 초기화 작업을 사용할 것입니다. `storeOp`에 대해서는 두 가지 가능성만 있습니다:

* `VK_ATTACHMENT_STORE_OP_STORE`: 렌더링된 내용이 메모리에 저장되어 나중에 읽을 수 있음
* `VK_ATTACHMENT_STORE_OP_DONT_CARE`: 렌더링 작업 후 프레임버퍼의 내용이 정의되지 않음

우리는 화면에 렌더링된 삼각형을 보고 싶으므로, 여기서는 저장 작업을 선택하겠습니다.

```c++
colorAttachment.stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
colorAttachment.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
```

`loadOp`와 `storeOp`는 컬러와 깊이 데이터에 적용되며, `stencilLoadOp`와 `stencilStoreOp`는 스텐실 데이터에 적용됩니다. 우리의 애플리케이션은 스텐실 버퍼를 사용하지 않을 것이므로, 로딩과 저장의 결과는 중요하지 않습니다.

```c++
colorAttachment.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
colorAttachment.finalLayout = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR;
```

Vulkan에서 텍스처와 프레임버퍼는 특정 픽셀 형식을 가진 `VkImage` 객체로 표현되지만, 메모리에서 픽셀의 레이아웃은 이미지로 하려는 작업에 따라 달라질 수 있습니다.

가장 일반적인 레이아웃들은 다음과 같습니다:

* `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL`: 컬러 어태치먼트로 사용되는 이미지
* `VK_IMAGE_LAYOUT_PRESENT_SRC_KHR`: 스왑 체인에서 표시될 이미지
* `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`: 메모리 복사 작업의 대상으로 사용될 이미지

이 주제에 대해서는 텍스처링 장에서 더 자세히 다룰 것입니다. 지금 알아야 할 중요한 점은 이미지들이 다음에 관여할 작업에 적합한 특정 레이아웃으로 전환되어야 한다는 것입니다.

`initialLayout`은 렌더 패스가 시작되기 전에 이미지가 가질 레이아웃을 지정합니다. `finalLayout`은 렌더 패스가 끝날 때 자동으로 전환될 레이아웃을 지정합니다. `initialLayout`에 `VK_IMAGE_LAYOUT_UNDEFINED`를 사용한다는 것은 이미지의 이전 레이아웃이 무엇이었는지 신경 쓰지 않는다는 의미입니다. 이 특별한 값의 주의사항은 이미지의 내용이 보존된다는 보장이 없다는 것이지만, 어차피 초기화할 것이므로 문제가 되지 않습니다. 렌더링 후에는 스왑 체인을 사용하여 표시할 수 있도록 이미지를 준비하고 싶으므로, `finalLayout`으로 `VK_IMAGE_LAYOUT_PRESENT_SRC_KHR`을 사용합니다.

## 서브패스와 어태치먼트 참조

하나의 렌더 패스는 여러 서브패스로 구성될 수 있습니다. 서브패스는 이전 패스의 프레임버퍼 내용에 의존하는 후속 렌더링 작업입니다. 예를 들어 하나씩 적용되는 일련의 후처리 효과들이 있습니다. 이러한 렌더링 작업들을 하나의 렌더 패스로 그룹화하면, Vulkan은 작업들을 재정렬하고 메모리 대역폭을 절약하여 더 나은 성능을 얻을 수 있습니다. 하지만 우리의 첫 번째 삼각형에서는 단일 서브패스만 사용하겠습니다.

각 서브패스는 이전 섹션에서 설명한 구조체를 사용하여 기술한 어태치먼트들 중 하나 이상을 참조합니다. 이러한 참조는 다음과 같은 `VkAttachmentReference` 구조체입니다:

```c++
VkAttachmentReference colorAttachmentRef{};
colorAttachmentRef.attachment = 0;
colorAttachmentRef.layout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
```

`attachment` 매개변수는 어태치먼트 설명 배열에서의 인덱스로 어떤 어태치먼트를 참조할지 지정합니다. 우리의 배열은 하나의 `VkAttachmentDescription`으로 구성되어 있으므로, 인덱스는 `0`입니다. `layout`은 이 참조를 사용하는 서브패스 동안 어태치먼트가 가지기를 원하는 레이아웃을 지정합니다. Vulkan은 서브패스가 시작될 때 자동으로 어태치먼트를 이 레이아웃으로 전환할 것입니다. 우리는 어태치먼트를 컬러 버퍼로 사용하려고 하며, 이름에서 알 수 있듯이 `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL` 레이아웃이 가장 좋은 성능을 제공할 것입니다.

서브패스는 `VkSubpassDescription` 구조체를 사용하여 설명됩니다:

```c++
VkSubpassDescription subpass{};
subpass.pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS;
```

Vulkan은 나중에 컴퓨트 서브패스도 지원할 수 있으므로, 이것이 그래픽스 서브패스라는 것을 명시적으로 지정해야 합니다. 다음으로, 컬러 어태치먼트에 대한 참조를 지정합니다:

```c++
subpass.colorAttachmentCount = 1;
subpass.pColorAttachments = &colorAttachmentRef;
```

이 배열에서의 어태치먼트 인덱스는 프래그먼트 셰이더에서 `layout(location = 0) out vec4 outColor` 지시어로 직접 참조됩니다!

서브패스에서 참조할 수 있는 다른 종류의 어태치먼트들은 다음과 같습니다:

* `pInputAttachments`: 셰이더에서 읽는 어태치먼트
* `pResolveAttachments`: 멀티샘플링 컬러 어태치먼트에 사용되는 어태치먼트
* `pDepthStencilAttachment`: 깊이와 스텐실 데이터를 위한 어태치먼트
* `pPreserveAttachments`: 이 서브패스에서는 사용되지 않지만 데이터를 보존해야 하는 어태치먼트

## 렌더 패스

이제 어태치먼트와 이를 참조하는 기본적인 서브패스가 설명되었으므로, 렌더 패스 자체를 생성할 수 있습니다. `pipelineLayout` 변수 바로 위에 `VkRenderPass` 객체를 담을 새로운 클래스 멤버 변수를 만드세요:

```c++
VkRenderPass renderPass;
VkPipelineLayout pipelineLayout;
```

그런 다음 어태치먼트와 서브패스의 배열을 포함하는 `VkRenderPassCreateInfo` 구조체를 채워서 렌더 패스 객체를 생성할 수 있습니다. `VkAttachmentReference` 객체들은 이 배열의 인덱스를 사용하여 어태치먼트를 참조합니다.

```c++
VkRenderPassCreateInfo renderPassInfo{};
renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO;
renderPassInfo.attachmentCount = 1;
renderPassInfo.pAttachments = &colorAttachment;
renderPassInfo.subpassCount = 1;
renderPassInfo.pSubpasses = &subpass;

if (vkCreateRenderPass(device, &renderPassInfo, nullptr, &renderPass) != VK_SUCCESS) {
    throw std::runtime_error("failed to create render pass!");
}
```

파이프라인 레이아웃과 마찬가지로, 렌더 패스는 프로그램 전체에서 참조될 것이므로 마지막에만 정리해야 합니다:

```c++
void cleanup() {
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    vkDestroyRenderPass(device, renderPass, nullptr);
    ...
}
```

많은 작업이었지만, 다음 장에서는 이 모든 것이 합쳐져서 마침내 그래픽스 파이프라인 객체를 생성하게 됩니다!

[C++ 코드](/code/11_render_passes.cpp) /
[버텍스 셰이더](/code/09_shader_base.vert) /
[프래그먼트 셰이더](/code/09_shader_base.frag)