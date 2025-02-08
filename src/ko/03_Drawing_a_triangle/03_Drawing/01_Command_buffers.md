# 커맨드 버퍼

Vulkan에서 드로우 작업이나 메모리 전송과 같은 명령은 함수 호출을 통해 직접 실행되지 않습니다. 수행하려는 모든 작업을 커맨드 버퍼 객체에 기록해야 합니다. 이 방식의 장점은 Vulkan에 우리가 원하는 작업을 전달할 준비가 되었을 때, 모든 명령이 함께 제출되므로 Vulkan이 명령을 더 효율적으로 처리할 수 있다는 것입니다. 또한, 이 방식은 원하는 경우 여러 스레드에서 명령 기록을 수행할 수 있도록 합니다.

## 커맨드 풀

커맨드 버퍼를 생성하기 전에 커맨드 풀을 생성해야 합니다. 커맨드 풀은 버퍼를 저장하는 데 사용되는 메모리를 관리하며, 커맨드 버퍼는 이 풀에서 할당됩니다. `VkCommandPool`을 저장할 새로운 클래스 멤버를 추가합니다:

```c++
VkCommandPool commandPool;
```

그런 다음 `createCommandPool`이라는 새로운 함수를 만들고, 프레임버퍼 생성 후에 `initVulkan`에서 호출합니다.

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
    createFramebuffers();
    createCommandPool();
}

...

void createCommandPool() {

}
```

커맨드 풀 생성에는 두 가지 매개변수만 필요합니다:

```c++
QueueFamilyIndices queueFamilyIndices = findQueueFamilies(physicalDevice);

VkCommandPoolCreateInfo poolInfo{};
poolInfo.sType = VK_STRUCTURE_TYPE_COMMAND_POOL_CREATE_INFO;
poolInfo.flags = VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT;
poolInfo.queueFamilyIndex = queueFamilyIndices.graphicsFamily.value();
```

커맨드 풀에는 두 가지 가능한 플래그가 있습니다:

* `VK_COMMAND_POOL_CREATE_TRANSIENT_BIT`: 커맨드 버퍼가 매우 자주 새로운 명령으로 다시 기록될 것임을 나타냅니다 (메모리 할당 동작을 변경할 수 있음).
* `VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT`: 커맨드 버퍼가 개별적으로 다시 기록될 수 있도록 합니다. 이 플래그가 없으면 모든 커맨드 버퍼를 함께 재설정해야 합니다.

우리는 매 프레임마다 커맨드 버퍼를 기록할 것이므로, 커맨드 버퍼를 재설정하고 다시 기록할 수 있어야 합니다. 따라서 `VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT` 플래그를 설정해야 합니다.

커맨드 버퍼는 그래픽스 큐나 프레젠테이션 큐와 같은 디바이스 큐에 제출하여 실행됩니다. 각 커맨드 풀은 단일 유형의 큐에 제출되는 커맨드 버퍼만 할당할 수 있습니다. 우리는 드로우 명령을 기록할 것이므로 그래픽스 큐 패밀리를 선택했습니다.

```c++
if (vkCreateCommandPool(device, &poolInfo, nullptr, &commandPool) != VK_SUCCESS) {
    throw std::runtime_error("failed to create command pool!");
}
```

`vkCreateCommandPool` 함수를 사용하여 커맨드 풀 생성을 완료합니다. 특별한 매개변수는 없습니다. 명령은 프로그램 전체에서 화면에 물체를 그리는 데 사용되므로, 풀은 프로그램 종료 시에만 파괴되어야 합니다:

```c++
void cleanup() {
    vkDestroyCommandPool(device, commandPool, nullptr);

    ...
}
```

## 커맨드 버퍼 할당

이제 커맨드 버퍼를 할당할 수 있습니다.

`VkCommandBuffer` 객체를 클래스 멤버로 생성합니다. 커맨드 버퍼는 해당 커맨드 풀이 파괴될 때 자동으로 해제되므로 명시적인 정리가 필요하지 않습니다.

```c++
VkCommandBuffer commandBuffer;
```

이제 `createCommandBuffer` 함수를 작업하여 커맨드 풀에서 단일 커맨드 버퍼를 할당합니다.

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
    createFramebuffers();
    createCommandPool();
    createCommandBuffer();
}

...

void createCommandBuffer() {

}
```

커맨드 버퍼는 `vkAllocateCommandBuffers` 함수를 사용하여 할당됩니다. 이 함수는 커맨드 풀과 할당할 버퍼의 수를 지정하는 `VkCommandBufferAllocateInfo` 구조체를 매개변수로 받습니다:

```c++
VkCommandBufferAllocateInfo allocInfo{};
allocInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
allocInfo.commandPool = commandPool;
allocInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
allocInfo.commandBufferCount = 1;

if (vkAllocateCommandBuffers(device, &allocInfo, &commandBuffer) != VK_SUCCESS) {
    throw std::runtime_error("failed to allocate command buffers!");
}
```

`level` 매개변수는 할당된 커맨드 버퍼가 주 커맨드 버퍼인지 보조 커맨드 버퍼인지를 지정합니다.

* `VK_COMMAND_BUFFER_LEVEL_PRIMARY`: 큐에 제출하여 실행할 수 있지만, 다른 커맨드 버퍼에서 호출할 수 없습니다.
* `VK_COMMAND_BUFFER_LEVEL_SECONDARY`: 직접 제출할 수는 없지만, 주 커맨드 버퍼에서 호출할 수 있습니다.

여기서는 보조 커맨드 버퍼 기능을 사용하지 않지만, 주 커맨드 버퍼에서 공통 작업을 재사용하는 데 유용할 수 있습니다.

우리는 단일 커맨드 버퍼만 할당할 것이므로 `commandBufferCount` 매개변수는 1입니다.

## 커맨드 버퍼 기록

이제 `recordCommandBuffer` 함수를 작업하여 실행하려는 명령을 커맨드 버퍼에 기록합니다. 사용할 `VkCommandBuffer`와 기록하려는 현재 스왑 체인 이미지의 인덱스가 매개변수로 전달됩니다.

```c++
void recordCommandBuffer(VkCommandBuffer commandBuffer, uint32_t imageIndex) {

}
```

커맨드 버퍼 기록을 시작할 때는 항상 `vkBeginCommandBuffer`를 호출하며, 이 함수는 해당 커맨드 버퍼의 사용에 대한 세부 정보를 지정하는 작은 `VkCommandBufferBeginInfo` 구조체를 인자로 받습니다.

```c++
VkCommandBufferBeginInfo beginInfo{};
beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
beginInfo.flags = 0; // 선택적
beginInfo.pInheritanceInfo = nullptr; // 선택적

if (vkBeginCommandBuffer(commandBuffer, &beginInfo) != VK_SUCCESS) {
    throw std::runtime_error("failed to begin recording command buffer!");
}
```

`flags` 매개변수는 커맨드 버퍼를 어떻게 사용할지 지정합니다. 다음 값들이 가능합니다:

* `VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT`: 커맨드 버퍼는 한 번 실행된 후 바로 다시 기록됩니다.
* `VK_COMMAND_BUFFER_USAGE_RENDER_PASS_CONTINUE_BIT`: 이는 단일 렌더 패스 내에서 완전히 실행되는 보조 커맨드 버퍼입니다.
* `VK_COMMAND_BUFFER_USAGE_SIMULTANEOUS_USE_BIT`: 커맨드 버퍼가 실행 중인 동안에도 다시 제출할 수 있습니다.

현재로서는 이러한 플래그 중 어느 것도 우리에게 적용되지 않습니다.

`pInheritanceInfo` 매개변수는 보조 커맨드 버퍼에만 관련이 있습니다. 이는 호출하는 주 커맨드 버퍼로부터 어떤 상태를 상속할지 지정합니다.

커맨드 버퍼가 이미 한 번 기록되었다면, `vkBeginCommandBuffer` 호출은 암시적으로 이를 재설정합니다. 나중에 버퍼에 명령을 추가하는 것은 불가능합니다.

## 렌더 패스 시작

드로우 작업은 `vkCmdBeginRenderPass`로 렌더 패스를 시작하는 것으로 시작됩니다. 렌더 패스는 `VkRenderPassBeginInfo` 구조체의 일부 매개변수를 사용하여 구성됩니다.

```c++
VkRenderPassBeginInfo renderPassInfo{};
renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO;
renderPassInfo.renderPass = renderPass;
renderPassInfo.framebuffer = swapChainFramebuffers[imageIndex];
```

첫 번째 매개변수는 렌더 패스 자체와 바인딩할 어태치먼트입니다. 우리는 각 스왑 체인 이미지에 대해 프레임버퍼를 생성했으며, 이는 컬러 어태치먼트로 지정되었습니다. 따라서 우리는 드로우할 스왑 체인 이미지에 해당하는 프레임버퍼를 바인딩해야 합니다. 전달된 `imageIndex` 매개변수를 사용하여 현재 스왑 체인 이미지에 맞는 프레임버퍼를 선택할 수 있습니다.

```c++
renderPassInfo.renderArea.offset = {0, 0};
renderPassInfo.renderArea.extent = swapChainExtent;
```

다음 두 매개변수는 렌더 영역의 크기를 정의합니다. 렌더 영역은 셰이더 로드 및 스토어가 발생할 위치를 정의합니다. 이 영역 밖의 픽셀은 정의되지 않은 값을 가집니다. 최상의 성능을 위해 어태치먼트의 크기와 일치해야 합니다.

```c++
VkClearValue clearColor = {{{0.0f, 0.0f, 0.0f, 1.0f}}};
renderPassInfo.clearValueCount = 1;
renderPassInfo.pClearValues = &clearColor;
```

마지막 두 매개변수는 컬러 어태치먼트의 로드 작업으로 `VK_ATTACHMENT_LOAD_OP_CLEAR`를 사용할 때 사용할 클리어 값을 정의합니다. 저는 클리어 색상을 단순히 검은색으로 100% 불투명도로 정의했습니다.

```c++
vkCmdBeginRenderPass(commandBuffer, &renderPassInfo, VK_SUBPASS_CONTENTS_INLINE);
```

이제 렌더 패스를 시작할 수 있습니다. 모든 명령 기록 함수는 `vkCmd` 접두사로 구분할 수 있습니다. 이 함수들은 모두 `void`를 반환하므로, 기록이 완료될 때까지 오류 처리가 없습니다.

모든 명령의 첫 번째 매개변수는 항상 명령을 기록할 커맨드 버퍼입니다. 두 번째 매개변수는 방금 제공한 렌더 패스의 세부 정보를 지정합니다. 마지막 매개변수는 렌더 패스 내에서 드로우 명령이 어떻게 제공될지 제어합니다. 이는 두 가지 값 중 하나를 가질 수 있습니다:

* `VK_SUBPASS_CONTENTS_INLINE`: 렌더 패스 명령이 주 커맨드 버퍼 자체에 포함되며, 보조 커맨드 버퍼가 실행되지 않습니다.
* `VK_SUBPASS_CONTENTS_SECONDARY_COMMAND_BUFFERS`: 렌더 패스 명령이 보조 커맨드 버퍼에서 실행됩니다.

우리는 보조 커맨드 버퍼를 사용하지 않을 것이므로 첫 번째 옵션을 선택합니다.

## 기본 드로우 명령

이제 그래픽스 파이프라인을 바인딩할 수 있습니다:

```c++
vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, graphicsPipeline);
```

두 번째 매개변수는 파이프라인 객체가 그래픽스 파이프라인인지 컴퓨트 파이프라인인지를 지정합니다. 이제 우리는 Vulkan에게 그래픽스 파이프라인에서 실행할 작업과 프래그먼트 셰이더에서 사용할 어태치먼트를 알려주었습니다.

[고정 함수 장](../02_Graphics_pipeline_basics/02_Fixed_functions.md#dynamic-state)에서 언급했듯이, 우리는 이 파이프라인에 대해 뷰포트와 가위 상태를 동적으로 지정했습니다. 따라서 드로우 명령을 실행하기 전에 커맨드 버퍼에서 이를 설정해야 합니다:

```c++
VkViewport viewport{};
viewport.x = 0.0f;
viewport.y = 0.0f;
viewport.width = static_cast<float>(swapChainExtent.width);
viewport.height = static_cast<float>(swapChainExtent.height);
viewport.minDepth = 0.0f;
viewport.maxDepth = 1.0f;
vkCmdSetViewport(commandBuffer, 0, 1, &viewport);

VkRect2D scissor{};
scissor.offset = {0, 0};
scissor.extent = swapChainExtent;
vkCmdSetScissor(commandBuffer, 0, 1, &scissor);
```

이제 삼각형을 그리기 위한 드로우 명령을 실행할 준비가 되었습니다:

```c++
vkCmdDraw(commandBuffer, 3, 1, 0, 0);
```

실제 `vkCmdDraw` 함수는 약간 실망스럽게도 매우 간단합니다. 하지만 이렇게 간단한 이유는 우리가 미리 지정한 모든 정보 때문입니다. 이 함수는 커맨드 버퍼 외에도 다음과 같은 매개변수를 가집니다:

* `vertexCount`: 버텍스 버퍼가 없더라도 기술적으로는 3개의 버텍스를 그릴 것입니다.
* `instanceCount`: 인스턴스 렌더링에 사용되며, 이를 사용하지 않는다면 `1`을 사용합니다.
* `firstVertex`: 버텍스 버퍼의 오프셋으로 사용되며, `gl_VertexIndex`의 최소값을 정의합니다.
* `firstInstance`: 인스턴스 렌더링의 오프셋으로 사용되며, `gl_InstanceIndex`의 최소값을 정의합니다.

## 마무리

이제 렌더 패스를 종료할 수 있습니다:

```c++
vkCmdEndRenderPass(commandBuffer);
```

그리고 커맨드 버퍼 기록을 완료합니다:

```c++
if (vkEndCommandBuffer(commandBuffer) != VK_SUCCESS) {
    throw std::runtime_error("failed to record command buffer!");
}
```

다음 장에서는 스왑 체인에서 이미지를 획득하고, 커맨드 버퍼를 기록 및 실행한 다음, 완료된 이미지를 스왑 체인에 반환하는 메인 루프 코드를 작성할 것입니다.

[C++ 코드](/code/14_command_buffers.cpp) /
[버텍스 셰이더](/code/09_shader_base.vert) /
[프래그먼트 셰이더](/code/09_shader_base.frag)
