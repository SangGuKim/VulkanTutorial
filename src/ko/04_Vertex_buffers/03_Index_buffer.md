# 인덱스 버퍼

## 소개

실제 애플리케이션에서 렌더링할 3D 메시는 종종 여러 삼각형 간에 버텍스를 공유합니다. 이는 사각형을 그릴 때와 같이 간단한 경우에도 이미 발생합니다:

![](/images/vertex_vs_index.svg)

사각형을 그리려면 두 개의 삼각형이 필요하므로, 6개의 버텍스가 필요한 버텍스 버퍼가 필요합니다. 문제는 두 버텍스의 데이터가 중복되어야 하며, 이는 50%의 중복을 초래합니다. 버텍스가 평균적으로 3개의 삼각형에서 재사용될 때 더 복잡한 메시에서는 상황이 더 악화됩니다. 이 문제의 해결책은 *인덱스 버퍼*를 사용하는 것입니다.

인덱스 버퍼는 본질적으로 버텍스 버퍼를 가리키는 포인터의 배열입니다. 이를 통해 버텍스 데이터의 순서를 재정렬하고 기존 데이터를 여러 버텍스에 재사용할 수 있습니다. 위의 그림은 각각의 네 개의 고유 버텍스를 포함하는 버텍스 버퍼가 있는 경우 사각형에 대한 인덱스 버퍼가 어떻게 생겼는지 보여줍니다. 처음 세 개의 인덱스는 오른쪽 위 삼각형을 정의하고, 마지막 세 개의 인덱스는 왼쪽 아래 삼각형의 버텍스를 정의합니다.

## 인덱스 버퍼 생성

이번 장에서는 버텍스 데이터를 수정하고 그림과 같은 사각형을 그리기 위해 인덱스 데이터를 추가할 것입니다. 네 모서리를 나타내도록 버텍스 데이터를 수정하세요:

```c++
const std::vector<Vertex> vertices = {
    {{-0.5f, -0.5f}, {1.0f, 0.0f, 0.0f}},
    {{0.5f, -0.5f}, {0.0f, 1.0f, 0.0f}},
    {{0.5f, 0.5f}, {0.0f, 0.0f, 1.0f}},
    {{-0.5f, 0.5f}, {1.0f, 1.0f, 1.0f}}
};
```

왼쪽 위 모서리는 빨간색, 오른쪽 위는 녹색, 오른쪽 아래는 파란색, 왼쪽 아래는 흰색입니다. `indices`라는 새 배열을 추가하여 인덱스 버퍼의 내용을 나타냅니다. 그림의 인덱스와 일치하여 오른쪽 위 삼각형과 왼쪽 아래 삼각형을 그려야 합니다.

```c++
const std::vector<uint16_t> indices = {
    0, 1, 2, 2, 3, 0
};
```

`vertices`의 항목 수에 따라 `uint16_t` 또는 `uint32_t`를 인덱스 버퍼에 사용할 수 있습니다. 지금은 65535개 미만의 고유 버텍스를 사용하고 있으므로 `uint16_t`를 사용할 수 있습니다.

버텍스 데이터처럼 인덱스도 GPU가 접근할 수 있도록 `VkBuffer`에 업로드해야 합니다. 인덱스 버퍼를 위한 리소스를 보관할 두 개의 새 클래스 멤버를 정의하세요:

```c++
VkBuffer vertexBuffer;
VkDeviceMemory vertexBufferMemory;
VkBuffer indexBuffer;
VkDeviceMemory indexBufferMemory;
```

이제 추가할 `createIndexBuffer` 함수는 `createVertexBuffer`와 거의 동일합니다:

```c++
void initVulkan() {
    ...
    createVertexBuffer();
    createIndexBuffer();
    ...
}

void createIndexBuffer() {
    VkDeviceSize bufferSize = sizeof(indices[0]) * indices.size();

    VkBuffer stagingBuffer;
    VkDeviceMemory stagingBufferMemory;
    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, stagingBuffer, stagingBufferMemory);

    void* data;
    vkMapMemory(device, stagingBufferMemory, 0, bufferSize, 0, &data);
    memcpy(data, indices.data(), (size_t) bufferSize);
    vkUnmapMemory(device, stagingBufferMemory);

    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_DST_BIT | VK_BUFFER_USAGE_INDEX_BUFFER_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, indexBuffer, indexBufferMemory);

    copyBuffer(stagingBuffer, indexBuffer, bufferSize);

    vkDestroyBuffer(device, stagingBuffer, nullptr);
    vkFreeMemory(device, stagingBufferMemory, nullptr);
}
```

두 가지 주목할만한 차이점이 있습니다. `bufferSize`는 이제 인덱스 수와 인덱스 유형의 크기, 즉 `uint16_t` 또는 `uint32_t`를 곱한 값과 같습니다. `indexBuffer`의 사용 용도는 `VK_BUFFER_USAGE_VERTEX_BUFFER_BIT` 대신 `VK_BUFFER_USAGE_INDEX_BUFFER_BIT`여야 합니다. 그 외에는 과정이 완전히 동일합니다. `indices`의 내용을 복사하기 위해 스테이징 버퍼를 생성한 다음 최종 디바이스 로컬 인덱스 버퍼로 복사합니다.

인덱스 버퍼는 프로그램이 끝날 때 버텍스 버퍼와 마찬가지로 정리되어야 합니다:

```c++
void cleanup() {
    cleanupSwapChain();

    vkDestroyBuffer(device, indexBuffer, nullptr);
    vkFreeMemory(device, indexBufferMemory, nullptr);

    vkDestroyBuffer(device, vertexBuffer, nullptr);
    vkFreeMemory(device, vertexBufferMemory, nullptr);

    ...
}
```

## 인덱스 버퍼 사용

인덱스 버퍼를 사용하여 그리기에는 `recordCommandBuffer`에 두 가지 변경이 필요합니다. 먼저 버텍스 버퍼와 마찬가지로 인덱스 버퍼를 바인드해야 합니다. 차이점은 인덱스 버퍼는 하나만 가질 수 있다는 것입니다. 각 버텍스 속성에 대해 다른 인덱스를 사용하는 것은 불가능하므로, 하나의 속성이라도 다르면 버텍스 데이터를 완전히 중복해야 합니다.

```c++
vkCmdBindVertexBuffers(commandBuffer, 0, 1, vertexBuffers, offsets);

vkCmdBindIndexBuffer(commandBuffer, indexBuffer, 0, VK_INDEX_TYPE_UINT16);
```

`vkCmdBindIndexBuffer`는 인덱스 버퍼, 그 안의 바이트 오프셋, 인덱스 데이터 유형을 매개변수로 사용하여 바인딩합니다. 이전에 언급했듯이 가능한 유형은 `VK_INDEX_TYPE_UINT16`과 `VK_INDEX_TYPE_UINT32`입니다.

인덱스 버퍼를 바인딩하는 것만으로는 아무 것도 변경되지 않으므로, Vulkan에 인덱스 버퍼를 사용하도록 지시하는 그리기 명령을 변경해야 합니다. `vkCmdDraw` 줄을 제거하고 `vkCmdDrawIndexed`로 대체하세요:

```c++
vkCmdDrawIndexed(commandBuffer, static_cast<uint32_t>(indices.size()), 1, 0, 0, 0);
```

이 함수 호출은 `vk

CmdDraw`와 매우 유사합니다. 첫 두 매개변수는 인덱스 수와 인스턴스 수를 지정합니다. 인스턴싱을 사용하지 않으므로 `1` 인스턴스를 지정하세요. 인덱스 수는 버텍스 셰이더에 전달될 버텍스 수를 나타냅니다. 다음 매개변수는 인덱스 버퍼의 오프셋을 지정하며, `1`의 값을 사용하면 그래픽 카드가 두 번째 인덱스에서 읽기를 시작합니다. 두 번째에서 마지막 매개변수는 인덱스 버퍼의 인덱스에 추가할 오프셋을 지정합니다. 마지막 매개변수는 인스턴싱에 대한 오프셋을 지정하는데, 우리는 사용하지 않습니다.

이제 프로그램을 실행하면 다음과 같은 모습을 볼 수 있습니다:

![](/images/indexed_rectangle.png)

이제 인덱스 버퍼를 사용하여 버텍스를 재사용함으로써 메모리를 절약하는 방법을 알게 되었습니다. 이는 복잡한 3D 모델을 로드할 예정인 미래의 장에서 특히 중요해질 것입니다.

이전 장에서는 여러 리소스를 단일 메모리 할당에서 할당해야 한다고 언급했지만, 실제로는 한 단계 더 나아가야 합니다. [드라이버 개발자들은](https://developer.nvidia.com/vulkan-memory-management) 버텍스 버퍼와 인덱스 버퍼와 같은 여러 버퍼를 단일 `VkBuffer`에 저장하고 `vkCmdBindVertexBuffers`와 같은 명령에서 오프셋을 사용하는 것이 데이터가 더 캐시 친화적이기 때문에 장점이 있다고 권장합니다. 심지어 같은 메모리 청크를 여러 리소스에 재사용할 수도 있습니다(물론 데이터를 새로 고친다는 전제 하에), 만약 그것들이 같은 렌더 작업 중에 사용되지 않는다면. 이는 *별칭(aliasing)*이라고 알려져 있으며, 일부 Vulkan 함수는 이를 수행하려는 의도를 명시적으로 지정할 수 있는 플래그를 가지고 있습니다.

[C++ 코드](/code/21_index_buffer.cpp) /
[버텍스 셰이더](/code/18_shader_vertexbuffer.vert) /
[프래그먼트 셰이더](/code/18_shader_vertexbuffer.frag)