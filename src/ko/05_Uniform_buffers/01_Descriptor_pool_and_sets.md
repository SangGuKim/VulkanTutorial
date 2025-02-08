# 디스크립터 풀 및 세트

## 소개

이전 장에서 설명한 디스크립터 세트 레이아웃은 바인딩될 수 있는 디스크립터의 유형을 설명합니다. 이번 장에서는 `VkBuffer` 리소스를 유니폼 버퍼 디스크립터에 바인딩하기 위해 디스크립터 세트를 생성할 것입니다.

## 디스크립터 풀

디스크립터 세트는 직접 생성할 수 없으며, 명령 버퍼처럼 풀에서 할당해야 합니다. 디스크립터 세트에 해당하는 것은 *디스크립터 풀*이라고 불립니다. `createDescriptorPool`이라는 새 함수를 작성하여 설정합니다.

```c++
void initVulkan() {
    ...
    createUniformBuffers();
    createDescriptorPool();
    ...
}

...

void createDescriptorPool() {

}
```

디스크립터 세트에 포함될 디스크립터 유형과 수를 `VkDescriptorPoolSize` 구조체를 사용하여 설명해야 합니다.

```c++
VkDescriptorPoolSize poolSize{};
poolSize.type = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
poolSize.descriptorCount = static_cast<uint32_t>(MAX_FRAMES_IN_FLIGHT);
```

우리는 매 프레임마다 이 디스크립터 중 하나를 할당할 것입니다. 이 풀 크기 구조는 메인 `VkDescriptorPoolCreateInfo`에서 참조됩니다:

```c++
VkDescriptorPoolCreateInfo poolInfo{};
poolInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO;
poolInfo.poolSizeCount = 1;
poolInfo.pPoolSizes = &poolSize;
```

개별 디스크립터뿐만 아니라 할당 가능한 최대 디스크립터 세트 수도 지정해야 합니다:

```c++
poolInfo.maxSets = static_cast<uint32_t>(MAX_FRAMES_IN_FLIGHT);
```

구조에는 명령 풀과 유사한 선택적 플래그가 있으며, 개별 디스크립터 세트를 자유롭게 해제할 수 있는지 여부를 결정합니다: `VK_DESCRIPTOR_POOL_CREATE_FREE_DESCRIPTOR_SET_BIT`. 디스크립터 세트를 생성한 후에는 수정하지 않을 예정이므로 이 플래그가 필요하지 않습니다. `flags`를 기본값 `0`으로 둘 수 있습니다.

```c++
VkDescriptorPool descriptorPool;

...

if (vkCreateDescriptorPool(device, &poolInfo, nullptr, &descriptorPool) != VK_SUCCESS) {
    throw std::runtime_error("failed to create descriptor pool!");
}
```

디스크립터 풀 핸들을 저장할 새 클래스 멤버를 추가하고 `vkCreateDescriptorPool`을 호출하여 생성합니다.

## 디스크립터 세트

이제 디스크립터 세트 자체를 할당할 수 있습니다. 이를 위해 `createDescriptorSets` 함수를 추가합니다:

```c++
void initVulkan() {
    ...
    createDescriptorPool();
    createDescriptorSets();
    ...
}

...

void createDescriptorSets() {

}
```

디스크립터 세트 할당은 `VkDescriptorSetAllocateInfo` 구조체로 설명됩니다. 할당할 디스크립터 풀, 할당할 디스크립터 세트 수 및 기준이 될 디스크립터 세트 레이아웃을 지정해야 합니다:

```c++
std::vector<VkDescriptorSetLayout> layouts(MAX_FRAMES_IN_FLIGHT, descriptorSetLayout);
VkDescriptorSetAllocateInfo allocInfo{};
allocInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO;
allocInfo.descriptorPool = descriptorPool;
allocInfo.descriptorSetCount = static_cast<uint32_t>(MAX_FRAMES_IN_FLIGHT);
allocInfo.pSetLayouts = layouts.data();
```

우리의 경우, 동일한 레이아웃을 가진 각 프레임을 위해 하나의 디스크립터 세트를 생성할 것입니다. 불행히도 같은 레이아웃의 복사본이 필요하기 때문에 다음 함수는 세트 수에 맞는 배열을 기대합니다.

디스크립터 세트 핸들을 저장할 클래스 멤버를 추가하고 `vkAllocateDescriptorSets`를 사용하여 할당합니다:

```c++
VkDescriptorPool descriptorPool;
std::vector<VkDescriptorSet> descriptorSets;

...

descriptorSets.resize(MAX_FRAMES_IN_FLIGHT);
if (vkAllocateDescriptorSets(device, &allocInfo, descriptorSets.data()) != VK_SUCCESS) {
    throw std::runtime_error("failed to allocate descriptor sets!");
}
```

디스크립터 세트를 명시적으로 정리할 필요는 없습니다. 왜냐하면 디스크립터 풀이 파괴될 때 자동으로 해제되기 때문입니다. `vkAllocateDescriptorSets` 호출은 유니폼 버퍼 디스크립터를 가진 디스크립터 세트를 할당할 것입니다.

```c++
void cleanup() {
    ...
    vkDestroyDescriptorPool(device, descriptorPool, nullptr);

    vkDestroyDescriptorSetLayout(device, descriptor

SetLayout, nullptr);
    ...
}
```

이제 디스크립터 세트가 할당되었지만, 디스크립터 내부는 아직 구성되지 않았습니다. 모든 디스크립터를 채우기 위해 루프를 추가합니다:

```c++
for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {

}
```

버퍼를 참조하는 디스크립터인 우리의 유니폼 버퍼 디스크립터는 `VkDescriptorBufferInfo` 구조체로 구성됩니다. 이 구조체는 디스크립터에 대한 데이터를 포함하는 버퍼와 그 영역을 명시합니다.

```c++
for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
    VkDescriptorBufferInfo bufferInfo{};
    bufferInfo.buffer = uniformBuffers[i];
    bufferInfo.offset = 0;
    bufferInfo.range = sizeof(UniformBufferObject);
}
```

전체 버퍼를 덮어쓰는 경우, `range`로 `VK_WHOLE_SIZE` 값을 사용할 수도 있습니다. 디스크립터의 구성은 `VkWriteDescriptorSet` 구조체의 배열을 매개변수로 취하는 `vkUpdateDescriptorSets` 함수를 사용하여 업데이트됩니다.

```c++
VkWriteDescriptorSet descriptorWrite{};
descriptorWrite.sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
descriptorWrite.dstSet = descriptorSets[i];
descriptorWrite.dstBinding = 0;
descriptorWrite.dstArrayElement = 0;
```

첫 두 필드는 업데이트할 디스크립터 세트와 바인딩을 명시합니다. 우리는 유니폼 버퍼 바인딩 인덱스 `0`을 사용했습니다. 디스크립터는 배열일 수 있으므로, 업데이트할 배열의 첫 인덱스도 지정해야 합니다. 우리는 배열을 사용하지 않으므로 인덱스는 간단히 `0`입니다.

```c++
descriptorWrite.descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
descriptorWrite.descriptorCount = 1;
```

다시 한번 디스크립터 유형을 명시해야 합니다. `dstArrayElement`에서 시작하는 배열에서 여러 디스크립터를 한 번에 업데이트할 수 있습니다. `descriptorCount` 필드는 업데이트하려는 배열 요소 수를 지정합니다.

```c++
descriptorWrite.pBufferInfo = &bufferInfo;
descriptorWrite.pImageInfo = nullptr; // Optional
descriptorWrite.pTexelBufferView = nullptr; // Optional
```

마지막 필드는 실제로 디스크립터를 구성하는 `descriptorCount` 구조체의 배열을 참조합니다. 디스크립터의 유형에 따라 세 가지 중 하나를 사용해야 합니다. `pBufferInfo` 필드는 버퍼 데이터를 참조하는 디스크립터에 사용됩니다, `pImageInfo`는 이미지 데이터를 참조하는 디스크립터에 사용되며, `pTexelBufferView`는 버퍼 뷰를 참조하는 디스크립터에 사용됩니다. 우리의 디스크립터는 버퍼를 기반으로 하므로 `pBufferInfo`를 사용합니다.

```c++
vkUpdateDescriptorSets(device, 1, &descriptorWrite, 0, nullptr);
```

업데이트는 `vkUpdateDescriptorSets`를 사용하여 적용됩니다. 이 함수는 매개변수로 `VkWriteDescriptorSet`과 `VkCopyDescriptorSet`의 두 가지 종류의 배열을 받습니다. 후자는 이름에서 알 수 있듯이 디스크립터를 서로 복사하는 데 사용됩니다.

## 디스크립터 세트 사용

이제 `recordCommandBuffer` 함수를 업데이트하여 각 프레임에 대해 적절한 디스크립터 세트를 셰이더의 디스크립터에 `vkCmdBindDescriptorSets`를 사용하여 실제로 바인딩해야 합니다. 이는 `vkCmdDrawIndexed` 호출 전에 수행되어야 합니다:

```c++
vkCmdBindDescriptorSets(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, pipelineLayout, 0, 1, &descriptorSets[currentFrame], 0, nullptr);
vkCmdDrawIndexed(commandBuffer, static_cast<uint32_t>(indices.size()), 1, 0, 0, 0);
```

버텍스 및 인덱스 버퍼와 달리, 디스크립터 세트는 그래픽 파이프라인에 고유하지 않습니다. 따라서 그래픽 또는 컴퓨트 파이프라인에 디스크립터 세트를 바인딩할 것인지 명시해야 합니다. 다음 매개변수는 디스크립터가 기반으로 하는 레이아웃입니다. 다음 세 매개변수는 첫 번째 디스크립터 세트의 인덱스, 바인딩할 세트 수 및 바인딩할 세트 배열을 지정합니다. 마지막 두 매개변수는 동적 디스크립터를 위한 오프셋 배열을 지정합니다. 이에 대해서는 향후 장에서 살펴볼 것입니다.

프로그램을 실행하면 아쉽게도 아무 것도 보이지 않는 것을 알 수 있습니다. 문제는 프로젝션 매트릭스에서 Y-플립을 수행했기 때문에 버텍스들이 시계 반대 방향으로 그려지고 있으며, 이로 인해 배면 제거가 발생하여 기하학적 도형이 그려지지 않기 때문입니다. `createGraphicsPipeline` 함수로 가서 `VkPipelineRasterizationStateCreateInfo`에서 `frontFace`를 수정하여 이를 바로잡으세요:

```c++
rasterizer.cullMode = VK_CULL_MODE_BACK_BIT;
rasterizer.frontFace = VK_FRONT_FACE_COUNTER_CLOCKWISE;
```

프로그램을 다시 실행하면 다음과 같은 결과를 볼 수 있습니다:

![](/images/spinning_square.png)

사각형이 정사각형으로 변했는데, 이는 프로젝션 매트릭스가 이제 종횡비를 고려하기 때문입니다. `updateUniformBuffer`는 화면 크기 조정을 처리하므로, `recreateSwapChain`에서 디스크립터 세트를 다시 만들 필요가 없습니다.

## 정렬 요구 사항

지금까지 C++ 구조체의 데이터가 셰이더의 유니폼 정의와 어떻게 일치해야 하는지에 대해 자세히 설명하지 않았습니다. 당연히 둘 다 동일한 유형을 사용하면 될 것 같습니다:

```c++
struct UniformBufferObject {
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};

layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;
```

하지만 그게 전부는 아닙니다. 예를 들어, 구조체와 셰이더를 다음과 같이 수정해 보세요:

```c++
struct UniformBufferObject {
    glm::vec2 foo;
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};

layout(binding = 0) uniform UniformBufferObject {
    vec2 foo;
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;
```

셰이더와 프로그램을 다시 컴파일하고 실행하면, 지금까지 작업했던 다채

로운 사각형이 사라진 것을 발견할 수 있습니다! 그 이유는 *정렬 요구 사항*을 고려하지 않았기 때문입니다.

Vulkan은 구조체의 데이터가 메모리에 특정 방식으로 정렬되어 있기를 요구합니다. 예를 들어:

- 스칼라는 N (= 32비트 float의 경우 4바이트)으로 정렬되어야 합니다.
- `vec2`는 2N (= 8바이트)으로 정렬되어야 합니다.
- `vec3` 또는 `vec4`는 4N (= 16바이트)으로 정렬되어야 합니다.
- 중첩 구조체는 멤버의 기본 정렬을 16의 배수로 올림해야 합니다.
- `mat4` 행렬은 `vec4`와 같은 정렬을 가져야 합니다.

이 정렬 요구 사항은 [명세](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap15.html#interfaces-resources-layout)에서 전체 목록을 찾을 수 있습니다.

원래 셰이더에 세 개의 `mat4` 필드만 있었기 때문에 정렬 요구 사항을 충족했습니다. 각 `mat4`는 4 x 4 x 4 = 64바이트 크기이므로, `model`의 오프셋은 `0`, `view`의 오프셋은 `64`, `proj`의 오프셋은 `128`입니다. 모두 16의 배수이므로 잘 작동했습니다.

새 구조체는 크기가 8바이트인 `vec2`로 시작하므로 모든 오프셋을 변경합니다. 이제 `model`의 오프셋은 `8`, `view`의 오프셋은 `72`, `proj`의 오프셋은 `136`이며, 이는 16의 배수가 아닙니다. 이 문제를 해결하려면 C++11에서 도입된 [`alignas`](https://en.cppreference.com/w/cpp/language/alignas) 지정자를 사용할 수 있습니다:

```c++
struct UniformBufferObject {
    glm::vec2 foo;
    alignas(16) glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};
```

이제 프로그램을 다시 컴파일하고 실행하면 셰이더가 행렬 값을 다시 올바르게 받는 것을 확인할 수 있습니다.

다행히도 대부분의 경우 이러한 정렬 요구 사항을 고려하지 않아도 됩니다. GLM을 포함하기 전에 `GLM_FORCE_DEFAULT_ALIGNED_GENTYPES`를 정의함으로써 GLM을 사용하여 `vec2`와 `mat4`의 정렬 요구 사항을 이미 지정한 버전을 사용할 수 있습니다:

```c++
#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEFAULT_ALIGNED_GENTYPES
#include <glm/glm.hpp>
```

이 정의를 추가하면 `alignas` 지정자를 제거할 수 있고 프로그램은 여전히 잘 작동해야 합니다.

그러나 중첩 구조체를 사용하기 시작하면 이 방법이 실패할 수 있습니다. 다음과 같은 C++ 코드에서 정의된 것을 고려하십시오:

```c++
struct Foo {
    glm::vec2 v;
};

struct UniformBufferObject {
    Foo f1;
    Foo f2;
};
```

그리고 다음과 같은 셰이더 정의:

```c++
struct Foo {
    vec2 v;
};

layout(binding = 0) uniform UniformBufferObject {
    Foo f1;
    Foo f2;
} ubo;
```

이 경우 `f2`는 오프셋 `8`에 있어야 하지만 중첩 구조체이므로 오프셋 `16`에 있어야 합니다. 이 경우 정렬을 직접 지정해야 합니다:

```c++
struct UniformBufferObject {
    Foo f1;
    alignas(16) Foo f2;
};
```

이런 식의 문제를 피하기 위해 항상 정렬을 명시하는 것이 좋습니다. 그렇게 하면 정렬 오류의 이상한 증상에 의해 놀라지 않게 됩니다.

```c++
struct UniformBufferObject {
    alignas(16) glm::mat4 model;
    alignas(16) glm::mat4 view;
    alignas(16) glm::mat4 proj;
};
```

`foo` 필드를 제거한 후 셰이더를 다시 컴파일하는 것을 잊지 마세요.

## 다중 디스크립터 세트

일부 구조와 함수 호출에서 암시된 것처럼, 실제로 동시에 여러 디스크립터 세트를 바인딩할 수 있습니다. 파이프라인 레이아웃을 생성할 때 각 디스크립터 세트에 대한 디스크립터 세트 레이아웃을 지정해야 합니다. 셰이더는 다음과 같이 특정 디스크립터 세트를 참조할 수 있습니다:

```c++
layout(set = 0, binding = 0) uniform UniformBufferObject { ... }
```

이 기능을 사용하여 객체별로 다르고 공유되는 디스크립터를 별도의 디스크립터 세트에 넣을 수 있습니다. 이 경우 대부분의 디스크립터를 그리기 호출 간에 다시 바인딩할 필요가 없으므로 효율적일 수 있습니다.

[C++ 코드](/code/23_descriptor_sets.cpp) /
[버텍스 셰이더](/code/22_shader_ubo.vert) /
[프래그먼트 셰이더](/code/22_shader_ubo.frag)