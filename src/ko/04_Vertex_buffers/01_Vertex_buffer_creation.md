# 버텍스 버퍼 생성

## 소개

Vulkan에서 버퍼는 그래픽 카드가 읽을 수 있는 임의의 데이터를 저장하는 메모리 영역입니다. 이번 장에서는 버텍스 데이터를 저장하는 데 사용하지만, 향후 장에서 탐구할 다른 많은 용도로도 사용될 수 있습니다. 지금까지 다루었던 Vulkan 객체와 달리 버퍼는 자체적으로 메모리를 자동으로 할당하지 않습니다. 이전 장에서 본 것처럼 Vulkan API는 프로그래머가 거의 모든 것을 제어하도록 하며, 메모리 관리도 그 중 하나입니다.

## 버퍼 생성

`initVulkan`에서 `createCommandBuffers` 바로 전에 호출할 새로운 함수 `createVertexBuffer`를 생성합니다.

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
    createVertexBuffer();
    createCommandBuffers();
    createSyncObjects();
}

...

void createVertexBuffer() {

}
```

버퍼를 생성하려면 `VkBufferCreateInfo` 구조체를 채워야 합니다.

```c++
VkBufferCreateInfo bufferInfo{};
bufferInfo.sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO;
bufferInfo.size = sizeof(vertices[0]) * vertices.size();
```

구조체의 첫 번째 필드는 `size`로, 버퍼의 크기를 바이트 단위로 지정합니다. 버텍스 데이터의 바이트 크기를 계산하는 것은 `sizeof`를 사용하여 간단합니다.

```c++
bufferInfo.usage = VK_BUFFER_USAGE_VERTEX_BUFFER_BIT;
```

두 번째 필드는 `usage`로, 버퍼의 데이터가 사용될 목적을 나타냅니다. 비트 연산자를 사용하여 여러 용도를 지정할 수 있습니다. 우리의 경우에는 버텍스 버퍼로 사용될 것이며, 향후 장에서 다른 유형의 용도를 살펴볼 것입니다.

```c++
bufferInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;
```

스왑 체인의 이미지처럼 버퍼도 특정 큐 패밀리가 소유하거나 동시에 여러 개와 공유될 수 있습니다. 버퍼는 그래픽 큐에서만 사용될 것이므로 독점적 접근을 유지할 수 있습니다.

`flags` 매개변수는 지금 당장 관련이 없는 희소 버퍼 메모리를 구성하는 데 사용됩니다. 기본값인 `0`으로 두겠습니다.

이제 `vkCreateBuffer`로 버퍼를 생성할 수 있습니다. 버퍼 핸들을 저장할 클래스 멤버를 정의하고 `vertexBuffer`라고 합니다.

```c++
VkBuffer vertexBuffer;

...

void createVertexBuffer() {
    VkBufferCreateInfo bufferInfo{};
    bufferInfo.sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO;
    bufferInfo.size = sizeof(vertices[0]) * vertices.size();
    bufferInfo.usage = VK_BUFFER_USAGE_VERTEX_BUFFER_BIT;
    bufferInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;

    if (vkCreateBuffer(device, &bufferInfo, nullptr, &vertexBuffer) != VK_SUCCESS) {
        throw std::runtime_error("failed to create vertex buffer!");
    }
}
```

버퍼는 프로그램이 끝날 때까지 렌더링 명령에서 사용할 수 있어야 하며, 스왑 체인에 의존하지 않으므로 원래 `cleanup` 함수에서 정리하겠습니다.

```c++
void cleanup() {
    cleanupSwapChain();

    vkDestroyBuffer(device, vertexBuffer, nullptr);

    ...
}
```

## 메모리 요구 사항

버퍼는 생성되었지만 아직 메모리가 할당되지 않았습니다. 버퍼에 메모리를 할당하는 첫 번째 단계는 `vkGetBufferMemoryRequirements` 함수를 사용하여 메모리 요구 사항을 조회하는 것입니다.

```c++
VkMemoryRequirements memRequirements;
vkGetBufferMemoryRequirements(device, vertexBuffer, &memRequirements);
```

`VkMemoryRequirements` 구조체는 세 개의 필드를 가집니다:

* `size`: 필요한 메모리 양의 크기를 바이트 단위로, `bufferInfo.size`와 다를 수 있습니다.
* `alignment`: 할당된 메모리 영역에서 버퍼가 시작하는 바이트 단위의 오프셋, `bufferInfo.usage` 및 `bufferInfo.flags`에 따라 다릅니다.
* `memoryTypeBits`: 버퍼에 적합한 메모리 유형의 비트 필드.

그래픽 카드는 할당할 수 있는 다양한 유형의 메모리를 제공할 수 있습니다. 각 메모리 유형은 허용된 작업과 성능 특성면에서 다릅니다. 버퍼의 요구 사항과 우리 자신의 애플리케이션 요구 사항을 결합하여 사용할 메모리 유형을 찾아야 합니다. 이 목적을 위해 새로운 함수 `findMemoryType`을 생성하겠습니다.

```c++
uint32_t findMemoryType(uint32_t typeFilter, VkMemoryPropertyFlags properties) {

}
```

먼저 `vkGetPhysicalDeviceMemoryProperties`를 사용하여 사용 가능한 메모리 유형에 대한 정보를 조회해야 합니다.

```c++
VkPhysicalDeviceMemoryProperties memProperties;
vkGetPhysicalDeviceMemoryProperties(physicalDevice, &memProperties);
```

`VkPhysicalDeviceMemoryProperties` 구조체에는 `memoryTypes` 및 `memoryHeaps` 두 개의 배열이 있습니다. 메모리 힙은 전용 VRAM과 VRAM이 부족할 때 사용되는 RAM의 스왑 공간과 같은 독립된 메모리 리소스입니다. 다양한 유형의 메모리는 이 힙 내에 존재합니다. 지금은 메모리 유형에만 관심을 가지고 힙에서 오는 것은 아니지만, 이것이 성능에 영향을 줄 수 있다고 상상할 수 있습니다.

우선 버퍼 자체에 적합한 메모리 유형을 찾겠습니다:

```c++
for (uint32_t i = 0; i < memProperties.memoryTypeCount; i++) {
    if (typeFilter & (1 << i)) {
        return i;
    }
}

throw std::runtime_error("failed to find suitable memory type!");
```

`typeFilter` 매개변수는 적합한 메모리 유형의 비트 필드를 지정하는 데 사용됩니다. 즉, 해당 비트가 `1`로 설정된 적합한 메모리 유형의 인덱스를 단순히 반복하여 확인할 수 있습니다.

하지만 우리는 버텍스 버퍼에 적합한 메모리 유형에만 관심이 있는 것이 아닙니다. 우리는 버텍스 데이터를 그 메모리에 쓸 수 있어야 합니다. `memoryTypes` 배열은 각 메모리 유형의 힙과 속성을 지정하는 `VkMemoryType` 구조체로 구성됩니다. 속성은 메모리의 특별한 기능, 예를 들어 CPU에서 쓸 수 있도록 매핑할 수 있는 기능을 정의합니다. 이 속성은 `VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT`로 나

타내지만, `VK_MEMORY_PROPERTY_HOST_COHERENT_BIT` 속성도 사용해야 합니다. 메모리를 매핑할 때 그 이유를 알게 될 것입니다.

이제 루프를 수정하여 이 속성을 지원하는지도 확인할 수 있습니다:

```c++
for (uint32_t i = 0; i < memProperties.memoryTypeCount; i++) {
    if ((typeFilter & (1 << i)) && (memProperties.memoryTypes[i].propertyFlags & properties) == properties) {
        return i;
    }
}
```

우리는 여러 원하는 속성을 가질 수 있으므로 비트 AND의 결과가 단순히 0이 아니라 원하는 속성 비트 필드와 같은지 확인해야 합니다. 버퍼에 적합하고 필요한 모든 속성을 가진 메모리 유형이 있다면 그 인덱스를 반환하고, 그렇지 않으면 예외를 발생시킵니다.

## 메모리 할당

이제 적합한 메모리 유형을 결정하는 방법이 있으므로 `VkMemoryAllocateInfo` 구조체를 채워 실제로 메모리를 할당할 수 있습니다.

```c++
VkMemoryAllocateInfo allocInfo{};
allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
allocInfo.allocationSize = memRequirements.size;
allocInfo.memoryTypeIndex = findMemoryType(memRequirements.memoryTypeBits, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT);
```

메모리 할당은 이제 크기와 유형을 지정하는 것만큼 간단하며, 두 값 모두 버텍스 버퍼의 메모리 요구 사항과 원하는 속성에서 파생됩니다. 메모리 핸들을 저장할 클래스 멤버를 생성하고 `vkAllocateMemory`로 할당합니다.

```c++
VkBuffer vertexBuffer;
VkDeviceMemory vertexBufferMemory;

...

if (vkAllocateMemory(device, &allocInfo, nullptr, &vertexBufferMemory) != VK_SUCCESS) {
    throw std::runtime_error("failed to allocate vertex buffer memory!");
}
```

메모리 할당에 성공하면 이제 이 메모리를 버퍼와 연결할 수 있습니다.

```c++
vkBindBufferMemory(device, vertexBuffer, vertexBufferMemory, 0);
```

첫 세 매개변수는 자명하며, 네 번째 매개변수는 메모리 영역 내의 오프셋입니다. 이 메모리는 버텍스 버퍼를 위해 특별히 할당되므로 오프셋은 단순히 `0`입니다. 오프셋이 0이 아닌 경우에는 `memRequirements.alignment`로 나눌 수 있어야 합니다.

물론 C++에서 동적 메모리 할당과 같이, 언젠가는 메모리를 해제해야 합니다. 버퍼 객체에 바인딩된 메모리는 버퍼가 더 이상 사용되지 않을 때 해제할 수 있습니다. 따라서 버퍼가 파괴된 후에 해제하겠습니다:

```c++
void cleanup() {
    cleanupSwapChain();

    vkDestroyBuffer(device, vertexBuffer, nullptr);
    vkFreeMemory(device, vertexBufferMemory, nullptr);
    ...
}
```

## 버텍스 버퍼 채우기

이제 버텍스 데이터를 버퍼에 복사할 시간입니다. 이는 `vkMapMemory`를 사용하여 버퍼 메모리를 CPU 접근 가능한 메모리에 매핑하여 수행됩니다.

```c++
void* data;
vkMapMemory(device, vertexBufferMemory, 0, bufferInfo.size, 0, &data);
```

이 함수를 사용하면 지정된 메모리 리소스의 오프셋과 크기로 정의된 영역에 접근할 수 있습니다. 여기서 오프셋과 크기는 각각 `0`과 `bufferInfo.size`입니다. 특별한 값 `VK_WHOLE_SIZE`를 사용하여 모든 메모리를 매핑할 수도 있습니다. 두 번째에서 마지막 매개변수는 현재 API에서 아직 사용할 수 없는 플래그를 지정하는 데 사용할 수 있습니다. 이는 `0`으로 설정해야 합니다. 마지막 매개변수는 매핑된 메모리에 대한 포인터를 출력으로 지정합니다.

```c++
void* data;
vkMapMemory(device, vertexBufferMemory, 0, bufferInfo.size, 0, &data);
memcpy(data, vertices.data(), (size_t) bufferInfo.size);
vkUnmapMemory(device, vertexBufferMemory);
```

이제 간단히 버텍스 데이터를 매핑된 메모리에 `memcpy`하여 다시 `vkUnmapMemory`를 사용하여 매핑을 해제할 수 있습니다. 불행히도 드라이버가 데이터를 버퍼 메모리에 즉시 복사하지 않을 수도 있습니다. 예를 들어 캐싱 때문입니다. 또한 매핑된 메모리에서 버퍼에 대한 쓰기가 아직 보이지 않을 수도 있습니다. 이 문제를 해결하는 두 가지 방법이 있습니다:

* 호스트 일관성이 있는 메모리 힙을 사용합니다. `VK_MEMORY_PROPERTY_HOST_COHERENT_BIT`으로 표시됩니다.
* 매핑된 메모리에 쓴 후 `vkFlushMappedMemoryRanges`를 호출하고, 매핑된 메모리에서 읽기 전에 `vkInvalidateMappedMemoryRanges`를 호출합니다.

우리는 첫 번째 접근 방식을 선택했습니다. 이는 매핑된 메모리가 항상 할당된 메모리의 내용과 일치하도록 보장합니다. 이 접근 방식이 명시적인 플러싱보다 약간 떨어지는 성능을 초래할 수도 있지만, 왜 그렇지 않은지 다음 장에서 살펴볼 것입니다.

메모리 범위를 플러싱하거나 일관된 메모리 힙을 사용하면 드라이버가 버퍼에 대한 우리의 쓰기를 인식하게 되지만, GPU에서 실제로 보이는 것은 아닙니다. 데이터를 GPU로 전송하는 작업은 배경에서 이루어지며, 사양은 다음 `vkQueueSubmit` 호출 시 완료된 것으로 [보장됩니다](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap7.html#synchronization-submission-host-writes).

## 버텍스 버퍼 바인딩

이제 남은 것은 렌더링 작업 중에 버텍스 버퍼를 바인딩하는 것입니다. 우리는 이를 수행하기 위해 `recordCommandBuffer` 함수를 확장할 것입니다.

```c++
vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, graphicsPipeline);

VkBuffer vertexBuffers[] = {vertexBuffer};
VkDeviceSize offsets[] = {0};
vkCmdBindVertexBuffers(commandBuffer, 0, 1, vertexBuffers, offsets);

vkCmdDraw(commandBuffer, static_cast<uint32_t>(vertices.size()), 1, 0, 0);
```

`vkCmdBindVertexBuffers` 함수는 이전 장에서 설정한 것처럼 버텍스 버퍼를 바인딩에 바인딩하는 데 사용됩니다. 명령 버퍼 외에 첫 두 매개변수는 바인딩에 대해 버텍스 버퍼를 지정할 오프셋과 수를 지정합니다. 마지막 두 매개변수는 바인딩할 버텍스 버퍼의

 배열과 버텍스 데이터를 읽기 시작할 바이트 오프셋을 지정합니다. 버퍼의 버텍스 수를 전달하기 위해 `vkCmdDraw` 호출도 변경해야 합니다.

이제 프로그램을 실행하면 익숙한 삼각형을 다시 볼 수 있습니다:

![](/images/triangle.png)

상단 버텍스의 색상을 `vertices` 배열을 수정하여 흰색으로 변경해 보세요:

```c++
const std::vector<Vertex> vertices = {
    {{0.0f, -0.5f}, {1.0f, 1.0f, 1.0f}},
    {{0.5f, 0.5f}, {0.0f, 1.0f, 0.0f}},
    {{-0.5f, 0.5f}, {0.0f, 0.0f, 1.0f}}
};
```

프로그램을 다시 실행하면 다음과 같은 모습을 볼 수 있습니다:

![](/images/triangle_white.png)

다음 장에서는 더 나은 성능을 제공하지만 조금 더 많은 작업이 필요한 다른 방식으로 버텍스 데이터를 버텍스 버퍼로 복사하는 방법을 살펴볼 것입니다.

[C++ 코드](/code/19_vertex_buffer.cpp) /
[버텍스 셰이더](/code/18_shader_vertexbuffer.vert) /
[프래그먼트 셰이더](/code/18_shader_vertexbuffer.frag)