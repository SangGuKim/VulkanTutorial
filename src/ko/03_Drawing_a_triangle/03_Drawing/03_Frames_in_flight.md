# 동시에 진행되는 프레임 (Frames in flight)

## 동시에 진행되는 프레임

현재 렌더 루프에는 한 가지 큰 결점이 있습니다. 다음 렌더링을 시작하기 전에 이전 프레임이 끝나기를 기다려야 하므로 호스트가 불필요하게 유휴 상태가 됩니다.

이를 해결하는 방법은 여러 프레임을 동시에 *진행 중*으로 허용하는 것입니다. 즉, 하나의 프레임 렌더링이 다음 프레임의 녹화와 상호 작용하지 않도록 허용하는 것입니다. 이를 어떻게 할까요? 렌더링 중에 접근하고 수정되는 모든 리소스는 중복되어야 합니다. 따라서 여러 명령 버퍼, 세마포어, 펜스가 필요합니다. 나중에 다른 리소스의 여러 인스턴스도 추가할 예정이므로 이 개념을 다시 보게 될 것입니다.

프로그램 상단에 동시에 처리할 프레임 수를 정의하는 상수를 추가하여 시작합니다:

```c++
const int MAX_FRAMES_IN_FLIGHT = 2;
```

CPU가 GPU보다 너무 앞서 나가지 않도록 하기 위해 숫자 2를 선택합니다. 2개의 프레임이 진행 중일 때, CPU와 GPU는 동시에 자신의 작업을 수행할 수 있습니다. CPU가 일찍 끝나면 GPU가 렌더링을 완료할 때까지 기다렸다가 더 많은 작업을 제출합니다. 3개 이상의 프레임이 진행 중이면 CPU가 GPU보다 앞서 나갈 수 있으며, 프레임 지연이 추가될 수 있습니다. 일반적으로 추가 지연은 원하지 않습니다. 하지만 응용 프로그램이 진행 중인 프레임 수를 제어할 수 있도록 하는 것은 Vulkan의 명시성의 또 다른 예입니다.

각 프레임은 자체 명령 버퍼, 세마포어 세트 및 펜스를 가져야 합니다. 이름을 변경한 다음 객체를 `std::vector`로 변경합니다:

```c++
std::vector<VkCommandBuffer> commandBuffers;

...

std::vector<VkSemaphore> imageAvailableSemaphores;
std::vector<VkSemaphore> renderFinishedSemaphores;
std::vector<VkFence> inFlightFences;
```

그런 다음 여러 명령 버퍼를 생성해야 합니다. `createCommandBuffer`를 `createCommandBuffers`로 이름을 변경합니다. 다음으로 명령 버퍼 벡터의 크기를 `MAX_FRAMES_IN_FLIGHT`의 크기로 조정하고, `VkCommandBufferAllocateInfo`를 그만큼의 명령 버퍼를 포함하도록 변경한 다음, 우리의 명령 버퍼 벡터로 대상을 변경해야 합니다:

```c++
void createCommandBuffers() {
    commandBuffers.resize(MAX_FRAMES_IN_FLIGHT);
    ...
    allocInfo.commandBufferCount = (uint32_t) commandBuffers.size();

    if (vkAllocateCommandBuffers(device, &allocInfo, commandBuffers.data()) != VK_SUCCESS) {
        throw std::runtime_error("failed to allocate command buffers!");
    }
}
```

`createSyncObjects` 함수는 모든 객체를 생성하도록 변경해야 합니다:

```c++
void createSyncObjects() {
    imageAvailableSemaphores.resize(MAX_FRAMES_IN_FLIGHT);
    renderFinishedSemaphores.resize(MAX_FRAMES_IN_FLIGHT);
    inFlightFences.resize(MAX_FRAMES_IN_FLIGHT);

    VkSemaphoreCreateInfo semaphoreInfo{};
    semaphoreInfo.sType = VK_STRUCTURE_TYPE_SEMAPHORE_CREATE_INFO;

    VkFenceCreateInfo fenceInfo{};
    fenceInfo.sType = VK_STRUCTURE_TYPE_FENCE_CREATE_INFO;
    fenceInfo.flags = VK_FENCE_CREATE_SIGNALED_BIT;

    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        if (vkCreateSemaphore(device, &semaphoreInfo, nullptr, &imageAvailableSemaphores[i]) != VK_SUCCESS ||
            vkCreateSemaphore(device, &semaphoreInfo, nullptr, &renderFinishedSemaphores[i]) != VK_SUCCESS ||
            vkCreateFence(device, &fenceInfo, nullptr, &inFlightFences[i]) != VK_SUCCESS) {

            throw std::runtime_error("failed to create synchronization objects for a frame!");
        }
    }
}
```

마찬가지로 모두 정리해야 합니다:

```c++
void cleanup() {
    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        vkDestroySemaphore(device, renderFinishedSemaphores[i], nullptr);
        vkDestroySemaphore(device, imageAvailableSemaphores[i], nullptr);
        vkDestroyFence(device, inFlightFences[i], nullptr);
    }

    ...
}
```

명령 버퍼는 명령 풀을 해제할 때 우리를 위해 자동으로 해제되므로 명령 버퍼 정리를 위해 추가로 할 일은 없습니다.

매 프레임마다 올바른 객체를 사용하려면 현재 프레임을 추적해야 합니다. 그 목적으로 프레임 인덱스를 사용할 것입니다:

```c++
uint32_t currentFrame = 0;
```

`drawFrame` 함수는 이제 올바른 객체를 사용하도록 수정할 수 있습니다:

```c++
void drawFrame() {
    vkWaitForFences(device, 1, &inFlightFences[currentFrame], VK_TRUE, UINT64_MAX);
    vkResetFences(device, 1, &inFlightFences[currentFrame]);

    vkAcquireNextImageKHR(device, swapChain, UINT64_MAX, imageAvailableSemaphores[currentFrame], VK_NULL_HANDLE, &imageIndex);

    ...

    vkResetCommandBuffer(commandBuffers[currentFrame],  0);
    recordCommandBuffer(commandBuffers[currentFrame], imageIndex);

    ...

    submitInfo.pCommandBuffers = &commandBuffers[currentFrame];

    ...

    VkSemaphore waitSemaphores[] = {imageAvailableSemaphores[currentFrame]};

    ...

    VkSemaphore signalSemaphores[] = {renderFinishedSemaphores[currentFrame]};

    ...

    if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, inFlightFences[currentFrame]) != VK_SUCCESS) {
}
```

물론, 매번 다음 프레임으로 진행하는 것을 잊지 말아야 합니다:

```c++
void drawFrame() {
    ...

    currentFrame = (currentFrame + 1) % MAX_FRAMES_IN_FLIGHT;
}
```

모듈로 (%) 연산자를 사용하여 `MAX_FRAMES_IN_FLIGHT`마다 대기열에 추가된 프레임 후에 프레임 인덱스가 반복되도록 합니다.

이제 `MAX_FRAMES_IN_FLIGHT` 이상의 프레임 작업이 대기열에 추가되지 않고 이러한 프레임이 서로 중첩되지 않도록 필요한 모든 동기화를 구현했습니다. 최종 정리와 같은 코드의 다른 부분이 `vkDeviceWaitIdle`과 같은 더 거친 동기화에 의존하는 것은 괜찮습니다. 어떤 접근 방식을 사용할지는 성능 요구 사항을 기준으로 결정해야 합니다.

동기화에 대해 자세히 알아보려면 Khronos의 [이 광범위한 개요](https://github.com/KhronosGroup/Vulkan-Docs/wiki/Synchronization-Examples#swapchain-image-acquire-and-present)를 살펴보세요.

다음 장에서는 Vulkan 프로그램이 잘 동작하기 위해 필요한 작은 것 하나를 처리할 것입니다.


[C++ 코드](/code/16_frames_in_flight.cpp) /
[버텍스 셰이더](/code/09_shader_base.vert) /
[프래그먼트 셰이더](/code/09_shader_base.frag)

