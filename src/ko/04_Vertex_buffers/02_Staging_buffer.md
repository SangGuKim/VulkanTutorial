# 스테이징 버퍼

## 소개

현재 우리가 가지고 있는 버텍스 버퍼는 정상적으로 작동하지만, CPU에서 접근할 수 있게 하는 메모리 유형이 그래픽 카드가 읽기에 가장 최적화된 메모리 유형은 아닐 수 있습니다. 가장 최적화된 메모리는 `VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT` 플래그가 설정되어 있으며, 일반적으로 전용 그래픽 카드에서 CPU가 접근할 수 없습니다. 이 장에서는 두 개의 버텍스 버퍼를 생성할 것입니다. 하나는 CPU 접근 가능 메모리에 있는 *스테이징 버퍼*로 버텍스 배열에서 데이터를 업로드하고, 최종 버텍스 버퍼는 디바이스 로컬 메모리에 있습니다. 데이터를 스테이징 버퍼에서 실제 버텍스 버퍼로 이동하기 위해 버퍼 복사 명령을 사용할 것입니다.

## 전송 큐

버퍼 복사 명령은 `VK_QUEUE_TRANSFER_BIT`를 지원하는 큐 패밀리가 필요합니다. 좋은 소식은 `VK_QUEUE_GRAPHICS_BIT` 또는 `VK_QUEUE_COMPUTE_BIT` 기능을 가진 모든 큐 패밀리가 이미 암시적으로 `VK_QUEUE_TRANSFER_BIT` 작업을 지원한다는 것입니다. 이 경우 `queueFlags`에 명시적으로 나열할 필요가 없습니다.

도전을 좋아한다면, 전송 작업을 위해 특별히 다른 큐 패밀리를 사용해 볼 수 있습니다. 이를 위해서는 프로그램을 다음과 같이 수정해야 합니다:

- `QueueFamilyIndices` 및 `findQueueFamilies`를 수정하여 `VK_QUEUE_TRANSFER_BIT`를 가지면서 `VK_QUEUE_GRAPHICS_BIT`는 가지지 않는 큐 패밀리를 명시적으로 찾습니다.
- `createLogicalDevice`를 수정하여 전송 큐 핸들을 요청합니다.
- 전송 큐 패밀리에서 제출된 명령 버퍼를 위한 두 번째 명령 풀을 생성합니다.
- 리소스의 `sharingMode`를 `VK_SHARING_MODE_CONCURRENT`로 변경하고 그래픽 및 전송 큐 패밀리를 모두 지정합니다.
- `vkCmdCopyBuffer`와 같은 모든 전송 명령을 그래픽 큐가 아닌 전송 큐에 제출합니다.

이 작업은 조금 번거롭지만 리소스가 큐 패밀리 간에 어떻게 공유되는지에 대해 많은 것을 배울 수 있습니다.

## 버퍼 생성 추상화

이 장에서 여러 버퍼를 생성할 예정이므로, 버퍼 생성을 도우미 함수로 옮기는 것이 좋습니다. `createBuffer`라는 새 함수를 생성하고 `createVertexBuffer`의 코드(매핑 제외)를 이동하세요.

```c++
void createBuffer(VkDeviceSize size, VkBufferUsageFlags usage, VkMemoryPropertyFlags properties, VkBuffer& buffer, VkDeviceMemory& bufferMemory) {
    VkBufferCreateInfo bufferInfo{};
    bufferInfo.sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO;
    bufferInfo.size = size;
    bufferInfo.usage = usage;
    bufferInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;

    if (vkCreateBuffer(device, &bufferInfo, nullptr, &buffer) != VK_SUCCESS) {
        throw std::runtime_error("failed to create buffer!");
    }

    VkMemoryRequirements memRequirements;
    vkGetBufferMemoryRequirements(device, buffer, &memRequirements);

    VkMemoryAllocateInfo allocInfo{};
    allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
    allocInfo.allocationSize = memRequirements.size;
    allocInfo.memoryTypeIndex = findMemoryType(memRequirements.memoryTypeBits, properties);

    if (vkAllocateMemory(device, &allocInfo, nullptr, &bufferMemory) != VK_SUCCESS) {
        throw std::runtime_error("failed to allocate buffer memory!");
    }

    vkBindBufferMemory(device, buffer, bufferMemory, 0);
}
```

버퍼 크기, 메모리 속성 및 사용 용도에 대한 매개변수를 추가하여 다양한 유형의 버퍼를 생성할 수 있도록 이 함수를 사용하십시오. 마지막 두 매개변수는 핸들을 쓸 출력 변수입니다.

이제 `createVertexBuffer`에서 버퍼 생성 및 메모리 할당 코드를 제거하고 대신 `createBuffer`를 호출할 수 있습니다:

```c++
void createVertexBuffer() {
    VkDeviceSize bufferSize = sizeof(vertices[0]) * vertices.size();
    createBuffer(bufferSize, VK_BUFFER_USAGE_VERTEX_BUFFER_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, vertexBuffer, vertexBufferMemory);

    void* data;
    vkMapMemory(device, vertexBufferMemory, 0, bufferSize, 0, &data);
        memcpy(data, vertices.data(), (size_t) bufferSize);
    vkUnmapMemory(device, vertexBufferMemory);
}
```

프로그램을 실행하여 버텍스 버퍼가 여전히 제대로 작동하는지 확인하세요.

## 스테이징 버퍼 사용

이제 `createVertexBuffer`를 수정하여 임시 버퍼로 호스트 가시 버퍼만 사용하고 실제 버텍스 버퍼로 디바이스 로컬 버퍼를 사용하도록 변경합니다.

```c++
void createVertexBuffer() {
    VkDeviceSize bufferSize = sizeof(vertices[0]) * vertices.size();

    VkBuffer stagingBuffer;
    VkDeviceMemory stagingBufferMemory;
    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, stagingBuffer, stagingBufferMemory);

    void* data;
    vkMapMemory(device, stagingBufferMemory, 0, bufferSize, 0, &data);
        memcpy(data, vertices.data(), (size_t) bufferSize);
    vkUnmapMemory(device, stagingBufferMemory);

    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_DST_BIT | VK_BUFFER_USAGE_VERTEX_BUFFER_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, vertexBuffer, vertexBufferMemory);
}
```

이제 버텍스 데이터를 매핑하고 복사하기 위해 `stagingBuffer`와 `stagingBufferMemory`를 사용합니다. 이 장에서는 두 개의 새로운 버퍼 사용 플래그를 사용합니다:

* `VK_BUFFER_USAGE_TRANSFER_SRC_BIT`: 버퍼를 메모리 전송 작업의 원본으로 사용할 수 있습니다.
* `VK_BUFFER_USAGE_TRANSFER_DST_BIT`: 버퍼를 메모리 전송 작업의 목적지로 사용할 수 있습니다.

`vertexBuffer`는 디바이스 로컬 메모리 유형에서 할당되며, 일반적으로 `vkMapMemory`를 사용할 수 없음을 의미합니다. 그러나 `stagingBuffer`에서 `vertexBuffer`로 데이터를 복사할 수 있습니다. `stagingBuffer`에 대해 전송 소스 플래그를 지정하고 `vertexBuffer`에 대해 전송 목적지 플래그와 버텍스 버퍼 사용 플래그를 지정함으로써 이를 수행하려는 의도를 나타내야 합니다.

이제 한 버퍼에서 다른 버퍼로 내용을 복사하는 함수 `copyBuffer`를 작성할 것입니다.

```c++
void copyBuffer(VkBuffer srcBuffer, VkBuffer dstBuffer, VkDeviceSize size) {

}
```

메모리 전송 작업은 그리기 명령과 마찬가지로 명령 버퍼를 사용하여 실행됩니다. 따라서 임시 명령 버퍼를 먼저 할당해야

 합니다. 이러한 종류의 단기 버퍼에 대해서는 명령 풀을 별도로 생성하는 것이 좋습니다. 구현은 메모리 할당 최적화를 적용할 수 있기 때문입니다. 그 경우 명령 풀 생성 중 `VK_COMMAND_POOL_CREATE_TRANSIENT_BIT` 플래그를 사용해야 합니다.

```c++
void copyBuffer(VkBuffer srcBuffer, VkBuffer dstBuffer, VkDeviceSize size) {
    VkCommandBufferAllocateInfo allocInfo{};
    allocInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
    allocInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
    allocInfo.commandPool = commandPool;
    allocInfo.commandBufferCount = 1;

    VkCommandBuffer commandBuffer;
    vkAllocateCommandBuffers(device, &allocInfo, &commandBuffer);
}
```

그리고 즉시 명령 버퍼 기록을 시작하세요:

```c++
VkCommandBufferBeginInfo beginInfo{};
beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
beginInfo.flags = VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT;

vkBeginCommandBuffer(commandBuffer, &beginInfo);
```

우리는 명령 버퍼를 한 번만 사용하고 복사 작업이 완료될 때까지 함수에서 반환되는 것을 기다릴 것입니다. 드라이버에 우리의 의도를 `VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT`을 사용하여 알리는 것이 좋습니다.

```c++
VkBufferCopy copyRegion{};
copyRegion.srcOffset = 0; // Optional
copyRegion.dstOffset = 0; // Optional
copyRegion.size = size;
vkCmdCopyBuffer(commandBuffer, srcBuffer, dstBuffer, 1, &copyRegion);
```

버퍼의 내용은 `vkCmdCopyBuffer` 명령을 사용하여 전송됩니다. 이 명령은 소스 버퍼와 목적지 버퍼를 인수로 받고 복사할 영역의 배열을 받습니다. 영역은 `VkBufferCopy` 구조체로 정의되며 소스 버퍼 오프셋, 목적지 버퍼 오프셋 및 크기로 구성됩니다. `vkMapMemory` 명령과 달리 여기서 `VK_WHOLE_SIZE`를 지정할 수 없습니다.

```c++
vkEndCommandBuffer(commandBuffer);
```

이 명령 버퍼에는 복사 명령만 포함되므로 그 후에 기록을 중지할 수 있습니다. 이제 명령 버퍼를 실행하여 전송을 완료하세요:

```c++
VkSubmitInfo submitInfo{};
submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;
submitInfo.commandBufferCount = 1;
submitInfo.pCommandBuffers = &commandBuffer;

vkQueueSubmit(graphicsQueue, 1, &submitInfo, VK_NULL_HANDLE);
vkQueueWaitIdle(graphicsQueue);
```

그리기 명령과 달리 이번에는 기다려야 할 이벤트가 없습니다. 우리는 버퍼의 전송을 즉시 실행하고 싶습니다. 이 전송이 완료되기를 기다리는 두 가지 방법이 있습니다. 펜스(fence)를 사용하고 `vkWaitForFences`로 기다리거나, 단순히 전송 큐가 유휴 상태가 될 때까지 기다리는 `vkQueueWaitIdle`을 사용할 수 있습니다. 펜스를 사용하면 여러 전송을 동시에 예약하고 모두 완료될 때까지 기다릴 수 있으며, 한 번에 하나씩 실행하는 대신 이렇게 하면 드라이버가 최적화할 기회를 더 많이 가질 수 있습니다.

```c++
vkFreeCommandBuffers(device, commandPool, 1, &commandBuffer);
```

전송 작업에 사용된 명령 버퍼를 정리하는 것을 잊지 마세요.

이제 `createVertexBuffer` 함수에서 `copyBuffer`를 호출하여 버텍스 데이터를 디바이스 로컬 버퍼로 이동할 수 있습니다:

```c++
createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_DST_BIT | VK_BUFFER_USAGE_VERTEX_BUFFER_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, vertexBuffer, vertexBufferMemory);

copyBuffer(stagingBuffer, vertexBuffer, bufferSize);
```

스테이징 버퍼에서 디바이스 버퍼로 데이터를 복사한 후에는 그것을 정리해야 합니다:

```c++
    ...

    copyBuffer(stagingBuffer, vertexBuffer, bufferSize);

    vkDestroyBuffer(device, stagingBuffer, nullptr);
    vkFreeMemory(device, stagingBufferMemory, nullptr);
}
```

프로그램을 실행하여 여전히 익숙한 삼각형을 볼 수 있는지 확인하세요. 개선 사항은 지금 당장 보이지 않을 수 있지만, 버텍스 데이터는 이제 고성능 메모리에서 로드됩니다. 이는 우리가 더 복잡한 기하학을 렌더링하기 시작할 때 중요해질 것입니다.

## 결론

실제 애플리케이션에서는 각각의 개별 버퍼에 대해 `vkAllocateMemory`를 호출해서는 안 됩니다. 동시 메모리 할당의 최대 수는 `maxMemoryAllocationCount` 물리적 디바이스 제한에 의해 제한되며, NVIDIA GTX 1080과 같은 고급 하드웨어에서는 `4096`으로 낮을 수 있습니다. 동시에 많은 객체에 대해 메모리를 할당하는 올바른 방법은 단일 할당을 많은 다른 객체들 사이에서 나누는 사용자 정의 할당자를 생성하는 것입니다. 이는 많은 함수에서 본 `offset` 매개변수를 사용합니다.

이러한 할당자를 직접 구현하거나 [VulkanMemoryAllocator](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator) 라이브러리를 사용할 수 있습니다. 이 라이브러리는 GPUOpen 이니셔티브에 의해 제공됩니다. 그러나 이 튜토리얼에서는 지금 당장 이러한 제한에 도달할 가능성이 없기 때문에 각 리소스에 대해 별도의 할당을 사용하는 것이 괜찮습니다.

[C++ 코드](/code/20_staging_buffer.cpp) /
[버텍스 셰이더](/code/18_shader_vertexbuffer.vert) /
[프래그먼트 셰이더](/code/18_shader_vertexbuffer.frag)