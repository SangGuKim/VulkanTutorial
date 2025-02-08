# 디스크립터 세트 레이아웃 및 버퍼

## 소개

우리는 이제 각 버텍스에 대해 버텍스 셰이더로 임의의 속성을 전달할 수 있지만, 
전역 변수는 어떨까요? 이 장부터 3D 그래픽으로 넘어가면서 모델-뷰-프로젝션 행렬이 
필요하게 됩니다. 이를 버텍스 데이터로 포함시킬 수 있지만, 이는 메모리 낭비이며 변환
(transform)이 변경될 때마다 버텍스 버퍼를 업데이트해야 합니다. 변환이 매 프레임마다 
쉽게 변경될 수 있습니다.

Vulkan에서 이 문제를 해결하는 올바른 방법은 *리소스 디스크립터*를 사용하는 것입니다.
디스크립터는 셰이더가 버퍼 및 이미지와 같은 리소스에 자유롭게 접근할 수 있는 방법입니다. 
변환 행렬을 포함하는 버퍼를 설정하고 버텍스 셰이더가 디스크립터를 통해 이에 접근하도록 
할 것입니다. 디스크립터의 사용은 세 부분으로 구성됩니다:

- 파이프라인 생성 중 디스크립터 세트 레이아웃 지정
- 디스크립터 풀에서 디스크립터 세트 할당
- 렌더링 중 디스크립터 세트 바인딩

*디스크립터 세트 레이아웃*은 파이프라인이 접근할 리소스 유형을 지정하며, 렌더 패스가 
접근할 첨부 유형을 지정하는 것과 비슷합니다. *디스크립터 세트*는 디스크립터에 바인딩될 
실제 버퍼 또는 이미지 리소스를 지정하며, 프레임버퍼가 렌더 패스 첨부에 바인딩할 실제 
이미지 뷰를 지정하는 것과 비슷합니다. 그런 다음 버텍스 버퍼와 프레임버퍼처럼 그리기 
명령에 디스크립터 세트가 바인딩됩니다.

이번 장에서는 유니폼 버퍼 객체(UBO)와 같은 디스크립터를 다룰 것입니다. 다른 유형의 
디스크립터는 추후 장에서 살펴볼 것이지만, 기본 과정은 동일합니다. 버텍스 셰이더가 
가지고 있기를 원하는 데이터를 C 구조체로 다음과 같이 가지고 있다고 가정해 봅시다:

```c++
struct UniformBufferObject {
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};
```

그런 다음 데이터를 `VkBuffer`에 복사하고 버텍스 셰이더에서 유니폼 버퍼 객체 디스크립터를 
통해 접근할 수 있습니다:

```glsl
layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```

이전 장에서 나온 사각형을 3D로 회전시켜서 매 프레임마다 모델, 뷰 및 프로젝션 행렬을 
업데이트할 것입니다.

네, 계속해서 전체 문서를 번역하겠습니다. 아래는 번역본입니다:

---

## 버텍스 셰이더

위에서 지정한 대로 유니폼 버퍼 객체를 포함하도록 버텍스 셰이더를 수정하세요. MVP 변환에 
익숙하다고 가정합니다. 그렇지 않다면 첫 장에서 언급된 [자료](https://www.opengl-tutorial.org/beginners-tutorials/tutorial-3-matrices/)를 
참조하세요.

```glsl
#version 450

layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;

layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;

layout(location = 0) out vec3 fragColor;

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```

`uniform`, `in` 및 `out` 선언의 순서는 중요하지 않습니다. `binding` 지시어는 속성에 
대한 `location` 지시어와 유사합니다. 이 바인딩을 디스크립터 세트 레이아웃에서 참조할 
것입니다. `gl_Position` 줄은 최종 위치를 클립 좌표로 계산하기 위해 변환을 사용하도록 
변경되었습니다. 2D 삼각형과 달리, 클립 좌표의 마지막 구성요소가 `1`이 아닐 수 있으며, 
최종 정규화된 디바이스 좌표로 변환될 때 나눗셈이 발생합니다. 이는 원근 분할(perspective 
division)로 사용되며, 더 가까운 객체가 더 멀리 있는 객체보다 크게 보이게 하는 데 필수적입니다.

## 디스크립터 세트 레이아웃

다음 단계는 C++ 측에서 UBO를 정의하고 버텍스 셰이더의 이 디스크립터에 대해 Vulkan에 
알리는 것입니다.

```c++
struct UniformBufferObject {
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};
```

GLM의 데이터 유형을 사용하여 셰이더의 정의와 정확히 일치하게 할 수 있습니다. 행렬의 데이터는 
셰이더가 기대하는 방식과 바이너리 호환되므로, 나중에 `UniformBufferObject`를 `VkBuffer`에 
`memcpy`할 수 있습니다.

셰이더에 사용된 모든 디스크립터 바인딩에 대한 세부 정보를 파이프라인 생성을 위해 제공해야 
합니다. 이는 모든 버텍스 속성과 그 `location` 인덱스를 해야 했던 것과 마찬가지입니다. 
이 모든 정보를 정의할 새로운 함수 `createDescriptorSetLayout`을 설정할 것입니다. 
파이프라인 생성 전에 호출해야 합니다. 왜냐하면 이 정보가 필요하기 때문입니다.

```c++
void initVulkan() {
    ...
    createDescriptorSetLayout();
    createGraphicsPipeline();
    ...
}

...

void createDescriptorSetLayout() {

}
```

모든 바인딩은 `VkDescriptorSetLayoutBinding` 구조체를 통해 설명되어야 합니다.

```c++
void createDescriptorSetLayout() {
    VkDescriptorSetLayoutBinding uboLayoutBinding{};
    uboLayoutBinding.binding = 0;
    uboLayoutBinding.descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
    uboLayoutBinding.descriptorCount = 1;
}
```

첫 두 필드는 셰이더에서 사용된 `binding`과 디스크립터의 유형을 지정하며, 여기서는 
유니폼 버퍼 객체입니다. 셰이더 변수가 유니폼 버퍼 객체의 배열을 나타낼 수 있으며, 
`descriptorCount`는 배열의 값 수를 지정합니다. 예를 들어, 골격 애니메이션에 대해 
각 뼈에 대한 변환을 지정하는 데 사용할 수 있습니다. 우리의 MVP 변환은 단일 유니폼 
버퍼 객체에 있으므로 `descriptorCount`는 `1`을 사용합니다.

```c++
uboLayoutBinding.stageFlags = VK_SHADER_STAGE_VERTEX_BIT;
```

디스크립터가 참조될 셰이더 스테이지를 지정해야 합니다. `stageFlags` 필드는 `VkShaderStageFlagBits` 
값의 조합이거나 `VK_SHADER_STAGE_ALL_GRAPHICS` 값일 수 있습니다. 우리의 경우에는 
버텍스 셰이더에서만 디스크립터를 참조합니다.

```c++
uboLayoutBinding.pImmutableSamplers =

 nullptr; // Optional
```

`pImmutableSamplers` 필드는 이미지 샘플링 관련 디스크립터에만 관련이 있으며, 나중에 
살펴볼 것입니다. 기본값으로 둘 수 있습니다.

모든 디스크립터 바인딩은 하나의 `VkDescriptorSetLayout` 객체로 결합됩니다. `pipelineLayout` 
위에 새 클래스 멤버를 정의하세요:

```c++
VkDescriptorSetLayout descriptorSetLayout;
VkPipelineLayout pipelineLayout;
```

`vkCreateDescriptorSetLayout`을 사용하여 생성할 수 있습니다. 이 함수는 바인딩 배열을 가진 
간단한 `VkDescriptorSetLayoutCreateInfo`를 수띍니다:

```c++
VkDescriptorSetLayoutCreateInfo layoutInfo{};
layoutInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO;
layoutInfo.bindingCount = 1;
layoutInfo.pBindings = &uboLayoutBinding;

if (vkCreateDescriptorSetLayout(device, &layoutInfo, nullptr, &descriptorSetLayout) != VK_SUCCESS) {
    throw std::runtime_error("failed to create descriptor set layout!");
}
```

파이프라인 생성 중에 디스크립터 세트 레이아웃을 지정해야 합니다. Vulkan이 셰이더가 사용할 
디스크립터를 알 수 있도록 하기 위해 파이프라인 레이아웃 객체에서 디스크립터 세트 레이아웃을 
참조해야 합니다. `VkPipelineLayoutCreateInfo`를 수정하여 레이아웃 객체를 참조하세요:

```c++
VkPipelineLayoutCreateInfo pipelineLayoutInfo{};
pipelineLayoutInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
pipelineLayoutInfo.setLayoutCount = 1;
pipelineLayoutInfo.pSetLayouts = &descriptorSetLayout;
```

여러 디스크립터 세트 레이아웃을 지정할 수 있는 이유가 궁금할 수 있습니다. 왜냐하면 하나의 
디스크립터 세트 레이아웃은 이미 모든 바인딩을 포함하기 때문입니다. 다음 장에서 디스크립터 
풀과 디스크립터 세트를 살펴볼 때 이에 대해 더 자세히 알아볼 것입니다.

디스크립터 세트 레이아웃은 프로그램이 종료될 때까지 새 그래픽 파이프라인을 생성할 수 있어야 
하므로 유지해야 합니다:

```c++
void cleanup() {
    cleanupSwapChain();

    vkDestroyDescriptorSetLayout(device, descriptorSetLayout, nullptr);

    ...
}
```

## 유니폼 버퍼

다음 장에서 셰이더가 이 변환 데이터에 접근할 수 있도록 `VkBuffer`를 유니폼 버퍼 디스크립터에 
실제로 바인딩하는 디스크립터 세트를 살펴볼 것입니다. 그러나 먼저 이 버퍼를 생성해야 합니다. 
매 프레임마다 새 데이터를 유니폼 버퍼에 복사할 것이므로 스테이징 버퍼를 사용하는 것은 의미가 
없습니다. 이 경우 추가 작업만 필요하며 성능을 저하시킬 수 있습니다.

동시에 진행 중인 여러 프레임이 있을 수 있으므로, 이전 프레임이 여전히 읽고 있는 동안 다음 
프레임을 준비하기 위해 버퍼를 업데이트하고 싶지 않기 때문에 프레임이 진행 중인 수만큼 
유니폼 버퍼를 가지고 있어야 합니다. 따라서 프레임이 진행 중인 수만큼 유니폼 버퍼가 있어야 
하며, GPU가 현재 읽고 있지 않은 유니폼 버퍼에 쓰기를 수행해야 합니다.

이를 위해 `uniformBuffers`, `uniformBuffersMemory`와 같은 새 클래스 멤버를 추가하세요:

```c++
VkBuffer indexBuffer;
VkDeviceMemory indexBufferMemory;

std::vector<VkBuffer> uniformBuffers;
std::vector<VkDeviceMemory> uniformBuffersMemory;
std::vector<void*> uniformBuffersMapped;
```

비슷하게, `createIndexBuffer` 후에 호출되고 버퍼를 할당하는 새 함수 `createUniformBuffers`를 생성하세요:

```c++
void initVulkan() {
    ...
    createVertexBuffer();
    createIndexBuffer();
    createUniformBuffers();
    ...
}

...

void createUniformBuffers() {
    VkDeviceSize bufferSize = sizeof(UniformBufferObject);

    uniformBuffers.resize(MAX_FRAMES_IN_FLIGHT);
    uniformBuffersMemory.resize(MAX_FRAMES_IN_FLIGHT);
    uniformBuffersMapped.resize(MAX_FRAMES_IN_FLIGHT);

    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        createBuffer(bufferSize, VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, uniformBuffers[i], uniformBuffersMemory[i]);

        vkMapMemory(device, uniformBuffersMemory[i], 0, bufferSize, 0, &uniformBuffersMapped[i]);
    }
}
```

생성 후 바로 `vkMapMemory`를 사용하여 나중에 데이터를 쓸 포인터를 얻습니다. 버퍼는 애플리케이션의 전체 수명 동안 이 포인터에 매핑되어 있습니다. 이 기법을 **"영구 매핑"**이라고 하며 모든 Vulkan 구현에서 작동합니다. 매번 업데이트할 필요 없이 매핑되지 않아 성능이 향상됩니다.

유니폼 데이터는 모든 그리기 호출에 사용되므로, 그것을 포함하는 버퍼는 렌더링을 중지할 때까지 제거되어서는 안 됩니다.

```c++
void cleanup() {
    ...

    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        vkDestroyBuffer(device, uniformBuffers[i], nullptr);
        vkFreeMemory(device, uniformBuffersMemory[i], nullptr);
    }

    vkDestroyDescriptorSetLayout(device, descriptorSetLayout, nullptr);

    ...

}
```

## 유니폼 데이터 업데이트

`drawFrame` 함수에서 다음 프레임을 제출하기 전에 새 함수 `updateUniformBuffer`를 호출하는 새 함수를 만드세요:

```c++
void drawFrame() {
    ...

    updateUniformBuffer(currentFrame);

    ...

    VkSubmitInfo submitInfo{};
    submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;

    ...
}

...

void updateUniformBuffer(uint32_t currentImage) {

}
```

이 함수는 매 프레임 기하학이 회전하도록 새 변환을 생성합니다. 이 기능을 구현하려면 두 개의 새 헤더를 포함해야 합니다:

```c++
#define GLM_FORCE_RADIANS
#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>

#include

 <chrono>
```

GLM은 각도를 라디안으로 기대합니다. `glm/gtc/matrix_transform.hpp`는 `glm::rotate` 및 `glm::perspective`와 같은 행렬 변환 함수를 제공합니다. 이제 함수에서 회전 및 투영 변환을 계산합니다:

```c++
void updateUniformBuffer(uint32_t currentImage) {
    static auto startTime = std::chrono::high_resolution_clock::now();

    auto currentTime = std::chrono::high_resolution_clock::now();
    float time = std::chrono::duration<float, std::chrono::seconds::period>(currentTime - startTime).count();

    UniformBufferObject ubo{};
    ubo.model = glm::rotate(glm::mat4(1.0f), time * glm::radians(90.0f), glm::vec3(0.0f, 0.0f, 1.0f));
    ubo.view = glm::lookAt(glm::vec3(2.0f, 2.0f, 2.0f), glm::vec3(0.0f, 0.0f, 0.0f), glm::vec3(0.0f, 0.0f, 1.0f));
    ubo.proj = glm::perspective(glm::radians(45.0f), swapChainExtent.width / (float) swapChainExtent.height, 0.1f, 10.0f);
    ubo.proj[1][1] *= -1;

    memcpy(uniformBuffersMapped[currentImage], &ubo, sizeof(ubo));
}
```

`startTime`은 첫 프레임이 렌더링되기 전에 저장됩니다. 이는 회전 변환을 계산하기 위해 타이머로 사용됩니다. `glm::rotate`는 주어진 각도로 주어진 축 주위에 4x4 변환 행렬을 생성합니다. `glm::lookAt` 함수는 뷰 변환을 생성합니다. 이는 시점, 초점 지점 및 "위쪽" 벡터를 사용합니다. 마지막으로, `glm::perspective`는 주어진 수직 시야각, 종횡비 및 깊이 범위를 가진 투영 변환을 생성합니다.

Vulkan은 클립 좌표에서 Y 좌표가 아래로 확장되도록 요구합니다. 그러나 GLM은 OpenGL을 기반으로 하며, 이는 Y 좌표가 위로 확장되도록 요구합니다. `ubo.proj[1][1]`에 `-1`을 곱하여 Y 좌표를 반전시켜 이를 수정합니다.

그리기 호출이 데이터에 접근하기 전에 매 프레임마다 유니폼 버퍼의 적절한 부분을 업데이트합니다.

## 결론

이 장에서는 유니폼 버퍼를 생성하고 매 프레임마다 그 내용을 업데이트하는 방법을 배웠습니다. 다음 장에서는 이 버퍼를 버텍스 셰이더에 연결하기 위해 필요한 디스크립터 세트를 할당하고 설정할 것입니다.

[C++ 코드](/code/22_descriptor_sets.cpp) /
[버텍스 셰이더](/code/20_shader_ubo.vert) /
[프래그먼트 셰이더](/code/20_shader_ubo.frag)