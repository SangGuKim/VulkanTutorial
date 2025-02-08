# 렌더링 및 프레젠테이션

이 장에서는 모든 것을 함께 조합할 것입니다. 메인 루프에서 호출되어 화면에 삼각형을 그리는 `drawFrame` 함수를 작성하겠습니다. 함수를 만들고 `mainLoop`에서 호출하는 것으로 시작합시다:

```c++
void mainLoop() {
    while (!glfwWindowShouldClose(window)) {
        glfwPollEvents();
        drawFrame();
    }
}

...

void drawFrame() {

}
```

## 프레임의 개요

Vulkan에서 프레임을 렌더링하는 것은 일반적으로 다음과 같은 단계로 이루어집니다:

* 이전 프레임이 완료될 때까지 대기
* 스왑 체인에서 이미지 획득
* 해당 이미지에 장면을 그리는 명령 버퍼 녹화
* 녹화된 명령 버퍼 제출
* 스왑 체인 이미지 프레젠테이션

이후 장에서 그리기 기능을 확장할 예정이지만, 지금은 이것이 렌더 루프의 핵심입니다.

## 동기화

Vulkan의 핵심 설계 철학은 GPU에서의 실행 동기화가 명시적이라는 것입니다. 작업의 실행 순서를 우리가 다양한 동기화 기본요소를 사용하여 정의합니다. 이는 많은 Vulkan API 호출이 비동기적으로 이루어진다는 것을 의미합니다. 함수는 작업이 완료되기 전에 반환됩니다.

이 장에서는 GPU에서 발생하는 여러 이벤트를 명시적으로 순서대로 정렬할 필요가 있습니다:

* 스왑 체인에서 이미지 획득
* 획득한 이미지에 그리기를 실행하는 명령 실행
* 그 이미지를 스크린에 표시하여 스왑체인에 반환

각각의 이벤트는 단일 함수 호출을 사용하여 시작되지만, 모두 비동기적으로 실행됩니다. 함수 호출은 작업이 실제로 완료되기 전에 반환되며 실행 순서 또한 정의되지 않습니다. 이는 불행한 일이며, 각 작업은 이전 작업이 완료되기를 필요로 합니다. 따라서 우리는 원하는 순서를 달성하기 위해 어떤 기본요소를 사용할 수 있는지 알아볼 필요가 있습니다.

### 세마포어

세마포어는 큐 작업 사이의 순서를 추가하는 데 사용됩니다. 큐 작업은 우리가 명령 버퍼에 제출하거나 나중에 보게 될 함수 내에서 제출하는 작업을 의미합니다. 예를 들어, 그래픽 큐와 프레젠테이션 큐 같은 큐가 있습니다. 세마포어는 동일한 큐 내의 작업과 다른 큐 간의 작업을 순서대로 할 때 사용됩니다.

세마포어에는 이진 세마포어와 타임라인 세마포어의 두 가지 유형이 있습니다. 이 튜토리얼에서는 이진 세마포어만 사용될 것이므로 타임라인 세마포어에 대해서는 논의하지 않겠습니다. 세마포어라는 용어는 이제 이진 세마포어만을 지칭합니다.

세마포어는 신호되지 않은 상태나 신호된 상태 중 하나입니다. 신호되지 않은 상태로 시작합니다. 세마포어를 큐 작업 사이에 순서를 추가하는 방법은 하나의 큐 작업에서 '신호' 세마포어로 제공하고 다른 큐 작업에서 '대기' 세마포어로 사용하는 것입니다. 예를 들어, 우리에게 세마포어 S가 있고 순서대로 실행하려는 큐 작업 A와 B가 있다고 가정해 봅시다. 우리가 Vulkan에게 말하는 것은 작업 A가 실행을 완료하면 세마포어 S를 '신호'하고, 작업 B는 실행을 시작하기 전에 세마포어 S에서 '대기'할 것입니다. 작업 A가 완료되면 세마포어 S는 신호될 것이며, 작업 B는 S가 신호되기 전까지 시작되지 않습니다. 작업 B가 실행을 시작한 후에는 세마포어 S가 자동으로 신호되지 않은 상태로 재설정되어 다시 사용될 수 있습니다.

### 펜스

펜스는 실행을 동기화하는 비슷한 목적을 가지고 있지만, CPU에서의 실행을 순서대로 하는 데 사용됩니다. 단순히 말해서, 호스트가 GPU가 무언가를 완료했는지 알아야 할 때 우리는 펜스를 사용합니다.

세마포어와 마찬가지로, 펜스는 신호되거나 신호되지 않은 상태 중 하나입니다. 우리가 실행할 작업을 제출할 때, 우리는 그 작업에 펜스를 첨부할 수 있습니다. 작업이 완료되면, 펜스는 신호됩니다. 그러면 우리는 호스트가 펜스가 신호될 때까지 기다리게 할 수 있습니다. 이는 호스트가 계속하기 전에 작업이 완료되었음을 보장합니다.

구체적인 예로, 스크린샷을 찍는 경우를 생각해 보겠습니다. GPU에서 필요한 작업을 이미 수행했다고 가정합니다. 이제 이미지를 GPU에서 호스트로 전송하고 그 메모리를 파일로 저장해야 합니다. 우리는 명령 버퍼 A와 펜스 F가 있는 전송을 실행하는 명령 버퍼 A를 제출합니다. 명령 버퍼 A를 펜스 F와 함께 제출한 후, 호스트가 F가 신호될 때까지 기다리라고 즉시 지시합니다. 이것은 호스트가 명령 버퍼 A의 실행이 완료될 때까지 차단됩니다. 따라서 우리는 메모리 전송이 완료되었으므로 호스트가 파일을 디스크에 저장하도록 허용할 수 있습니다.

## 동기화 객체 생성

이미지가 스왑 체인에서 획득되었음을 신호하는 세마포어, 렌더링이 완료되었고 프레젠테이션이 발생할 수 있음을 신호하는 또 다른 세마포어, 그리고 한 번에 하나의 프레임만 렌더링되도록 하는 펜스가 필요합니다.

이 세마포어 객체와 펜스 객체를 저장할 세 개의 클래스 멤버를 만듭니다:



```c++
VkSemaphore imageAvailableSemaphore;
VkSemaphore renderFinishedSemaphore;
VkFence inFlightFence;
```

세마포어를 만드는 데는 `VkSemaphoreCreateInfo`를 채워 넣어야 하지만, 현재 API 버전에서는 `sType` 외에 실제로 필요한 필드가 없습니다:

```c++
void createSyncObjects() {
    VkSemaphoreCreateInfo semaphoreInfo{};
    semaphoreInfo.sType = VK_STRUCTURE_TYPE_SEMAPHORE_CREATE_INFO;
}
```

펜스를 만드는 데는 `VkFenceCreateInfo`를 채워 넣어야 합니다:

```c++
VkFenceCreateInfo fenceInfo{};
fenceInfo.sType = VK_STRUCTURE_TYPE_FENCE_CREATE_INFO;
```

세마포어와 펜스를 만드는 것은 `vkCreateSemaphore` 및 `vkCreateFence`를 사용하는 친숙한 패턴을 따릅니다:

```c++
if (vkCreateSemaphore(device, &semaphoreInfo, nullptr, &imageAvailableSemaphore) != VK_SUCCESS ||
    vkCreateSemaphore(device, &semaphoreInfo, nullptr, &renderFinishedSemaphore) != VK_SUCCESS ||
    vkCreateFence(device, &fenceInfo, nullptr, &inFlightFence) != VK_SUCCESS) {
    throw std::runtime_error("failed to create semaphores!");
}
```

프로그램이 끝날 때, 모든 명령이 완료되고 더 이상 동기화가 필요 없을 때, 세마포어와 펜스를 정리해야 합니다:

```c++
void cleanup() {
    vkDestroySemaphore(device, imageAvailableSemaphore, nullptr);
    vkDestroySemaphore(device, renderFinishedSemaphore, nullptr);
    vkDestroyFence(device, inFlightFence, nullptr);
}
```

## 이전 프레임을 기다리기

프레임의 시작에서 이전 프레임이 완료될 때까지 기다리고 싶습니다. 그래서 명령 버퍼와 세마포어를 사용할 수 있습니다. 이를 위해 `vkWaitForFences`를 호출합니다:

```c++
void drawFrame() {
    vkWaitForFences(device, 1, &inFlightFence, VK_TRUE, UINT64_MAX);
}
```

`vkWaitForFences` 함수는 펜스 배열을 사용하며, 모든 펜스가 신호될 때까지 호스트가 기다립니다. 여기서 전달하는 `VK_TRUE`는 모든 펜스를 기다리겠다는 의미이지만, 단일 펜스의 경우는 상관없습니다. 이 함수는 또한 64비트 부호 없는 정수의 최대값인 `UINT64_MAX`를 타임아웃 매개변수로 사용하여 실질적으로 타임아웃을 비활성화합니다.

기다린 후에는 `vkResetFences` 호출을 통해 수동으로 펜스를 신호되지 않은 상태로 재설정해야 합니다:
```c++
    vkResetFences(device, 1, &inFlightFence);
```

진행하기 전에 우리 설계에 약간의 문제가 있습니다. `drawFrame()`을 처음 호출할 때, 즉시 `inFlightFence`가 신호될 때까지 기다립니다. `inFlightFence`는 프레임이 렌더링을 완료한 후에만 신호됩니다. 그러나 이것이 첫 번째 프레임이기 때문에 신호를 줄 이전 프레임이 없습니다! 따라서 `vkWaitForFences()`는 결코 일어나지 않을 일을 기다리며 무기한 차단됩니다.

이 딜레마에 대한 해결책은 많지만, API에 내장된 영리한 해결책이 있습니다. 펜스를 신호된 상태로 생성하여 첫 번째 `vkWaitForFences()` 호출이 즉시 반환되도록 합니다.

이를 수행하려면 `VK_FENCE_CREATE_SIGNALED_BIT` 플래그를 `VkFenceCreateInfo`에 추가합니다:

```c++
void createSyncObjects() {
    ...

    VkFenceCreateInfo fenceInfo{};
    fenceInfo.sType = VK_STRUCTURE_TYPE_FENCE_CREATE_INFO;
    fenceInfo.flags = VK_FENCE_CREATE_SIGNALED_BIT;

    ...
}
```

## 스왑 체인에서 이미지 획득

`drawFrame` 함수에서 다음으로 해야 할 일은 스왑 체인에서 이미지를 획득하는 것입니다. 스왑 체인은 확장 기능이므로 `vk*KHR` 명명 규칙을 사용하는 함수를 사용해야 합니다:

```c++
void drawFrame() {
    ...

    uint32_t imageIndex;
    vkAcquireNextImageKHR(device, swapChain, UINT64_MAX, imageAvailableSemaphore, VK_NULL_HANDLE, &imageIndex);
}
```

`vkAcquireNextImageKHR`의 첫 두 매개변수는 이미지를 획득하려는 논리적 장치와 스왑 체인입니다. 세 번째 매개변수는 이미지가 사용 가능해질 때까지의 타임아웃을 나노초 단위로 지정합니다. 64비트 부호 없는 정수의 최대값을 사용하면 타임아웃을 사실상 비활성화합니다.

다음 두 매개변수는 프레젠테이션 엔진이 이미지 사용을 완료했을 때 신호되는 동기화 객체를 지정합니다. 그때부터 우리는 그것에 그릴 수 있습니다. 세마포어, 펜스 또는 둘 다를 지정할 수 있습니다. 여기서는 그 목적으로 `imageAvailableSemaphore`를 사용합니다.

마지막 매개변수는 사용 가능해진 스왑 체인 이미지의 인덱스를 출력하는 변수를 지정합니다. 인덱스는 우리 `swapChainImages` 배열의 `VkImage`를 참조합니다. 우리는 그 인덱스를 사용하여 `VkFrameBuffer`를 선택합니다.

## 명령 버퍼 녹화

imageIndex로 사용할 스왑 체인 이미지를 지정하면 이제 명령 버퍼를 녹화할 수 있습니다. 먼저 명령 버퍼를 녹화할 수 있도록 `vkResetCommandBuffer`를 호출합니다.

```c++
vkResetCommandBuffer(commandBuffer, 0);
```

`vkResetCommandBuffer`의 두 번째 매개변수는 `VkCommandBufferResetFlagBits` 플래그입니다. 우리는 특별한 것을 원하지 않으므로 0으로 두겠습니다.

이제 `recordCommandBuffer` 함수를 호출하여 우리가 원하는 명령을 녹화하세요.

```c++
recordCommandBuffer(commandBuffer, imageIndex);
```

완전히 녹화된 명령 버퍼를 가지고 나면 이제 제출할 수 있습니다.

## 명령 버퍼 제출

큐 제출과 동기화는 `VkSubmitInfo` 구조체의 매개변수를 통해 구성됩니다.

```c++
VkSubmitInfo submitInfo{};
submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;

VkSemaphore waitSemaphores[] = {imageAvailableSemaphore};
VkPipelineStageFlags waitStages[] = {VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT};
submitInfo.waitSemaphoreCount = 1;
submitInfo.pWaitSemaphores = waitSemaphores;
submitInfo.pWaitDstStageMask = waitStages;
```

첫 세 매개변수는 실행이 시작되기 전에 어떤 세마포어를 기다릴지와 파이프라인의 어떤 단계에서 기다릴지를 지정합니다. 우리는 이미지가 사용 가능할 때까지 색상을 이미지에 쓰는 것을 기다리고

 싶습니다. 그래서 우리는 그래픽 파이프라인의 색상 첨부 단계를 지정합니다. 이는 이론적으로 구현이 이미지를 사용할 수 있기 전에 우리의 버텍스 셰이더 등을 이미 실행할 수 있음을 의미합니다. `waitStages` 배열의 각 항목은 `pWaitSemaphores`의 동일한 인덱스의 세마포어와 대응합니다.

```c++
submitInfo.commandBufferCount = 1;
submitInfo.pCommandBuffers = &commandBuffer;
```

다음 두 매개변수는 실제로 실행을 제출할 명령 버퍼를 지정합니다. 우리는 단순히 우리가 가진 단일 명령 버퍼를 제출합니다.

```c++
VkSemaphore signalSemaphores[] = {renderFinishedSemaphore};
submitInfo.signalSemaphoreCount = 1;
submitInfo.pSignalSemaphores = signalSemaphores;
```

`signalSemaphoreCount`와 `pSignalSemaphores` 매개변수는 명령 버퍼(들) 실행이 완료되면 신호할 세마포어를 지정합니다. 우리 경우에는 그 목적으로 `renderFinishedSemaphore`를 사용합니다.

```c++
if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, inFlightFence) != VK_SUCCESS) {
    throw std::runtime_error("failed to submit draw command buffer!");
}
```

이제 `vkQueueSubmit`을 사용하여 명령 버프를 그래픽 큐에 제출할 수 있습니다. 이 함수는 훨씬 더 큰 워크로드일 때 효율성을 위해 `VkSubmitInfo` 구조체 배열을 인자로 받습니다. 마지막 매개변수는 명령 버퍼가 실행을 완료할 때 신호될 선택적 펜스를 참조합니다. 이를 통해 명령 버퍼를 재사용할 때 안전하다는 것을 알 수 있습니다. 이제 다음 프레임에서 CPU는 이 명령 버퍼가 실행을 완료할 때까지 기다립니다.

## 서브패스 의존성

렌더 패스에서 서브패스는 자동으로 이미지 레이아웃 전환을 처리합니다. 이러한 전환은 *서브패스 의존성*을 통해 제어됩니다. 서브패스 의존성은 서브패스 간의 메모리 및 실행 의존성을 지정합니다. 우리는 지금 단 하나의 서브패스만 가지고 있지만, 이 서브패스 바로 전후의 작업도 암시적인 "서브패스"로 간주됩니다.

렌더 패스의 시작과 끝에서 전환을 처리하는 두 개의 내장된 의존성이 있지만, 전자는 적절한 시기에 발생하지 않습니다. 전환은 파이프라인의 시작에서 발생한다고 가정하지만, 그 시점에는 아직 이미지를 획득하지 않았습니다! 이 문제를 해결하는 두 가지 방법이 있습니다. `imageAvailableSemaphore`에 대한 `waitStages`를 `VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT`로 변경하여 렌더 패스가 이미지를 사용할 수 있을 때까지 시작되지 않도록 할 수 있습니다. 또는 렌더 패스가 `VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT` 단계에서 대기하도록 할 수 있습니다. 저는 여기서 두 번째 옵션을 선택했습니다. 이것은 서브패스 의존성과 그 작동 방식을 살펴볼 좋은 기회이기 때문입니다.

서브패스 의존성은 `VkSubpassDependency` 구조체에 지정됩니다. `createRenderPass` 함수로 가서 하나를 추가하세요:

```c++
VkSubpassDependency dependency{};
dependency.srcSubpass = VK_SUBPASS_EXTERNAL;
dependency.dstSubpass = 0;
```

첫 두 필드는 의존성과 의존된 서브패스의 인덱스를 지정합니다. 특별한 값 `VK_SUBPASS_EXTERNAL`은 `srcSubpass` 또는 `dstSubpass`에 지정되는 것에 따라 렌더 패스 전후의 암시적 서브패스를 참조합니다. 인덱스 `0`은 우리의 서브패스를 참조하며, 이것은 첫 번째이자 유일한 것입니다. `dstSubpass`는 항상 `srcSubpass`보다 높아야 합니다. 이는 의존성 그래프에서 순환을 방지하기 위해서입니다(하나의 서브패스가 `VK_SUBPASS_EXTERNAL`인 경우 제외).

```c++
dependency.srcStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
dependency.srcAccessMask = 0;
```

다음 두 필드는 어떤 작업을 기다리고 이 작업이 발생하는 단계를 지정합니다. 스왑 체인이 이미지에서 읽기를 완료할 때까지 기다려야 합니다. 이는 색상 첨부 출력 단계 자체를 기다리는 것으로 달성할 수 있습니다.

```c++
dependency.dstStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
dependency.dstAccessMask = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;
```

이 작업을 기다려야 할 작업은 색상 첨부 단계에 있으며 색상 첨부를 쓰는 것을 포함합니다. 이 설정은 전환이 실제로 필요하고 허용될 때까지 발생하지 않도록 방지합니다(즉, 우리가 그것에 색상을 시작하고 싶을 때).

```c++
renderPassInfo.dependencyCount = 1;
renderPassInfo.pDependencies = &dependency;
```

`VkRenderPassCreateInfo` 구조체는 의존성 배열을 지정하는 두 필드를 가지고 있습니다.

## 프레젠테이션

프레임을 그리는 마지막 단계는 결과를 스왑 체인에 다시 제출하여 결국 화면에 표시되도록 하는 것입니다. 프레젠테이션은 `drawFrame` 함수의 끝에서 `VkPresentInfoKHR` 구조체를 통해 구성됩니다.

```c++
VkPresentInfoKHR presentInfo{};
presentInfo.sType = VK_STRUCTURE_TYPE_PRESENT_INFO_KHR;

presentInfo.waitSemaphoreCount = 1;
presentInfo.pWaitSemaphores = signalSemaphores;
```

첫 두 매개변수는 프레젠테이션이 발생할 수 있기 전에 기다려야 할 세마포어를 지정합니다. `VkSubmitInfo`와 마찬가지입니다. 우리는 명령 버퍼가 실행을 완료하고, 따라서 우리의 삼각형이 그려지기를 기다리고 싶기 때문에, 신호될 세마포어를 가져와서 그것들을 기다리고, 따라서 `signalSemaphores`를 사용합니다.


```c++
VkSwapchainKHR swapChains[] = {swapChain};
presentInfo.swapchainCount = 1;
presentInfo.pSwapchains = swapChains;
presentInfo.pImageIndices = &imageIndex;
```

다음 두 매개변수는 이미지를 표시할 스왑 체인과 각 스왑 체인의 이미지 인덱스를 지정합니다. 거의 항상 하나일 것입니다.

```c++
presentInfo.pResults = nullptr; // 선택 사항
```

마지

막으로, `pResults`라는 선택적 매개변수가 있습니다. 이 매개변수를 사용하면 개별 스왑 체인마다 프레젠테이션이 성공했는지 확인할 수 있는 `VkResult` 값 배열을 지정할 수 있습니다. 단일 스왑 체인을 사용하는 경우에는 필요하지 않습니다. 왜냐하면 프레젠트 함수의 반환 값만 사용할 수 있기 때문입니다.

```c++
vkQueuePresentKHR(presentQueue, &presentInfo);
```

`vkQueuePresentKHR` 함수는 스왑 체인에 이미지를 표시하도록 요청을 제출합니다. `vkAcquireNextImageKHR` 및 `vkQueuePresentKHR`에 대한 오류 처리는 다음 장에서 추가할 것입니다. 왜냐하면 이 함수들의 실패는 지금까지 본 함수들과 달리 프로그램이 종료되어야 함을 의미하지 않기 때문입니다.

지금까지 모든 것을 올바르게 수행했다면, 프로그램을 실행할 때 다음과 같은 것을 볼 수 있습니다:

![](/images/triangle.png)

>이 색상 삼각형은 그래픽 튜토리얼에서 보통 보는 것과 다를 수 있습니다. 이 튜토리얼은 셰이더가 선형 색 공간에서 보간하고 그 후에 sRGB 색 공간으로 변환하도록 허용하기 때문입니다. 차이에 대한 논의는 [이 블로그 게시물](https://medium.com/@heypete/hello-triangle-meet-swift-and-wide-color-6f9e246616d9)을 참조하세요.

이제 유효성 검사 레이어가 활성화되어 있으면 프로그램이 종료될 때 충돌합니다. `debugCallback`에서 터미널로 출력된 메시지는 그 이유를 알려줍니다:

![](/images/semaphore_in_use.png)

`drawFrame`의 모든 작업이 비동기적이라는 것을 기억하세요. 그러므로 `mainLoop`에서 루프를 종료할 때, 그리기와 프레젠테이션 작업이 여전히 진행 중일 수 있습니다. 그 상황에서 리소스를 정리하는 것은 좋은 생각이 아닙니다.

이 문제를 해결하려면 `mainLoop`를 종료하고 창을 파괴하기 전에 논리적 장치가 작업을 완료할 때까지 기다려야 합니다:

```c++
void mainLoop() {
    while (!glfwWindowShouldClose(window)) {
        glfwPollEvents();
        drawFrame();
    }

    vkDeviceWaitIdle(device);
}
```

특정 명령 큐에서 작업이 완료될 때까지 기다리는 데 `vkQueueWaitIdle`을 사용할 수도 있습니다. 이 함수들은 동기화를 수행하는 매우 기초적인 방법으로 사용될 수 있습니다. 이제 창을 닫을 때 프로그램이 문제없이 종료됨을 볼 수 있습니다.

## 결론

약 900 줄이 넘는 코드 끝에, 우리는 드디어 화면에 무언가가 나타나는 단계에 도달했습니다! Vulkan 프로그램을 부트스트랩하는 것은 확실히 많은 작업이 필요하지만, 얻을 수 있는 메시지는 Vulkan이 명시성을 통해 엄청난 양의 제어를 제공한다는 것입니다. 이제 프로그램의 모든 Vulkan 객체의 목적과 서로의 관계에 대한 정신적 모델을 구축하기 위해 코드를 다시 읽는 시간을 가지는 것이 좋습니다. 이 지식을 기반으로 프로그램의 기능을 확장하기 시작할 것입니다.

다음 장은 렌더 루프를 확장하여 동시에 여러 프레임을 처리할 수 있도록 할 것입니다.

[C++ 코드](/code/15_hello_triangle.cpp) /
[버텍스 셰이더](/code/09_shader_base.vert) /
[프래그먼트 셰이더](/code/09_shader_base.frag)