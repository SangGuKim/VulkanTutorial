# 결합 이미지 샘플러 

## 소개

유니폼 버퍼 파트에서 처음으로 디스크립터에 대해 살펴보았습니다. 이 장에서는 새로운 유형의 디스크립터인 *결합 이미지 샘플러*를 살펴볼 것입니다. 이 디스크립터는 샘플러 객체를 통해 셰이더에서 이미지 자원에 접근할 수 있게 해줍니다. 이전 장에서 생성한 것과 같은 샘플러를 사용합니다.

먼저 디스크립터 세트 레이아웃, 디스크립터 풀 및 디스크립터 세트를 수정하여 이러한 결합 이미지 샘플러 디스크립터를 포함시키겠습니다. 그 후, `Vertex`에 텍스처 좌표를 추가하고 프래그먼트 셰이더를 수정하여 버텍스 색상을 단순히 보간하는 대신 텍스처에서 색상을 읽어올 것입니다.

## 디스크립터 업데이트

`createDescriptorSetLayout` 함수로 이동하여 결합 이미지 샘플러 디스크립터에 대한 `VkDescriptorSetLayoutBinding`을 추가하세요. 유니폼 버퍼 바로 다음 바인딩에 간단히 넣을 수 있습니다.

```c++
VkDescriptorSetLayoutBinding samplerLayoutBinding{};
samplerLayoutBinding.binding = 1;
samplerLayoutBinding.descriptorCount = 1;
samplerLayoutBinding.descriptorType = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
samplerLayoutBinding.pImmutableSamplers = nullptr;
samplerLayoutBinding.stageFlags = VK_SHADER_STAGE_FRAGMENT_BIT;

std::array<VkDescriptorSetLayoutBinding, 2> bindings = {uboLayoutBinding, samplerLayoutBinding};
VkDescriptorSetLayoutCreateInfo layoutInfo{};
layoutInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO;
layoutInfo.bindingCount = static_cast<uint32_t>(bindings.size());
layoutInfo.pBindings = bindings.data();
```

프래그먼트 셰이더에서 결합 이미지 샘플러 디스크립터를 사용하려는 의도를 나타내기 위해 `stageFlags`를 설정하세요. 프래그먼트의 색상이 결정되는 곳이기 때문입니다. 예를 들어, [높이맵](https://en.wikipedia.org/wiki/Heightmap)을 사용하여 버텍스 그리드를 동적으로 변형하기 위해 버텍스 셰이더에서 텍스처 샘플링을 사용할 수도 있습니다.

결합 이미지 샘플러를 위한 `VkPoolSize`를 추가하여 디스크립터 풀을 확장해야 합니다. `createDescriptorPool` 함수로 이동하여 이 디스크립터에 대한 `VkDescriptorPoolSize`를 포함시키도록 수정하세요.

```c++
std::array<VkDescriptorPoolSize, 2> poolSizes{};
poolSizes[0].type = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
poolSizes[0].descriptorCount = static_cast<uint32_t>(MAX_FRAMES_IN_FLIGHT);
poolSizes[1].type = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
poolSizes[1].descriptorCount = static_cast<uint32_t>(MAX_FRAMES_IN_FLIGHT);

VkDescriptorPoolCreateInfo poolInfo{};
poolInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO;
poolInfo.poolSizeCount = static_cast<uint32_t>(poolSizes.size());
poolInfo.pPoolSizes = poolSizes.data();
poolInfo.maxSets = static_cast<uint32_t>(MAX_FRAMES_IN_FLIGHT);
```

부적절한 디스크립터 풀은 검증 레이어가 잡아내지 못하는 좋은 예입니다: Vulkan 1.1부터 `vkAllocateDescriptorSets`는 풀이 충분히 크지 않으면 오류 코드 `VK_ERROR_POOL_OUT_OF_MEMORY`로 실패할 수 있습니다. 그러나 드라이버는 때때로 이 문제를 내부적으로 해결하려고 시도할 수 있습니다. 이는 때때로(하드웨어, 풀 크기 및 할당 크기에 따라 다름) 드라이버가 디스크립터 풀의 한계를 초과하는 할당을 허용하지만, 다른 경우에는 `vkAllocateDescriptorSets`가 실패하고 `VK_ERROR_POOL_OUT_OF_MEMORY`를 반환합니다. 이는 일부 기기에서는 할당이 성공하지만 다른 기기에서 실패할 때 특히 좌절스러울 수 있습니다.

Vulkan은 드라이버에 할당 책임을 이전함으로써, 디스크립터 풀 생성시 명시된 해당 `descriptorCount` 멤버에 따라 특정 유형의 디스크립터(`VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER` 등)를 할당하는 것이 엄격한 요구 사항이 아닙니다. 그러나 이는 여전히 최선의 방법이며, 향후 [최선의 실천 검증](https://vulkan.lunarg.com/doc/view/1.4.304.0/linux/best_practices.html)을 활성화하면 `VK_LAYER_KHRONOS_validation`이 이러한 유형의 문제에 대해 경고할 것입니다.

디스크립터 세트에 실제 이미지와 샘플러 자원을 바인딩하는 것이 마지막 단계입니다. `createDescriptorSets` 함수로 이동하세요.

```c++
for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
    VkDescriptorBufferInfo bufferInfo{};
    bufferInfo.buffer = uniformBuffers[i];
    bufferInfo.offset = 0;
    bufferInfo.range = sizeof(UniformBufferObject);

    VkDescriptorImageInfo imageInfo{};
    imageInfo.imageLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
    imageInfo.imageView = textureImageView;
    imageInfo.sampler = textureSampler;

    ...
}
```

결합 이미지 샘플러 구조체에 대한 자원은 `VkDescriptorImageInfo` 구조체에 명시해야 하며, 유니폼 버퍼 디스크립터의 버퍼 자원이 `VkDescriptorBufferInfo` 구조체에 명시되는 것과 같은 방식입니다. 이전 장의 객체들이 여기서 결합됩니다.

```c++
std::array<VkWriteDescriptorSet, 2> descriptorWrites{};

descriptorWrites[0].sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
descriptorWrites[0].dstSet = descriptorSets[i];
descriptorWrites[0].dstBinding = 0;
descriptorWrites[0].dstArrayElement = 0;
descriptorWrites[0].descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
descriptorWrites[0].descriptorCount = 1;
descriptorWrites[0].pBufferInfo = &bufferInfo;

descriptorWrites[1].sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
descriptorWrites[1].dstSet = descriptorSets[i];
descriptorWrites[1].dstBinding = 1;
descriptorWrites[1].dstArrayElement = 0;
descriptorWrites[1].descriptorType = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
descriptorWrites[1].descriptorCount = 1;
descriptorWrites[1].pImageInfo = &imageInfo;

vkUpdateDescriptorSets(device, static_cast<uint32_t>(descriptorWrites.size()), descriptorWrites.data(), 0, nullptr);
```

버퍼처럼 이 이미지 정보로 디스크립터를 업데이트해야 합니다. 이번에는 `pBufferInfo` 배열 대신 `pImageInfo` 배열을 사용합니다. 이제 디스크립터가 셰이더에 의해 사용될 준비가 되었습니다!

## 텍스처 좌표

텍스처 매핑에 필요한 중요한 구성 요소가 하나 더 있으며, 그것은 각 버텍스에 대한 실제 텍스처 좌표입니다. 텍스처 좌표는 이미지가 기하학적으로 어떻게 매핑되는지를 결정합니다.

```c++
struct Vertex {
    glm::vec2 pos;
    glm::vec3 color;
    glm::vec2 texCoord;

    static VkVertexInputBindingDescription getBindingDescription() {
        VkVertexInputBindingDescription bindingDescription{};
        bindingDescription.binding = 0;
        bindingDescription.stride = sizeof(Vertex);
        bindingDescription.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;

        return bindingDescription;
    }

    static std::array<VkVertexInputAttributeDescription, 3> getAttributeDescriptions() {
        std::array<VkVertexInputAttributeDescription, 3> attributeDescriptions{};

        attributeDescriptions[0].binding = 0;
        attributeDescriptions[0].location = 0;
        attributeDescriptions[0].format = VK_FORMAT_R32G32_SFLOAT;
        attributeDescriptions[0].offset = offsetof(Vertex, pos);

        attributeDescriptions[1].binding = 0;
        attributeDescriptions[1].location = 1;
        attributeDescriptions[1].format = VK_FORMAT_R32G32B32_SFLOAT;
        attributeDescriptions[1].offset = offsetof(Vertex, color);

        attributeDescriptions[2].binding = 0;
        attributeDescriptions[2].location = 2;
        attributeDescriptions[2].format = VK_FORMAT_R32G32_SFLOAT;
        attributeDescriptions[2].offset = offsetof(Vertex, texCoord);

        return attributeDescriptions;
    }
};
```

`Vertex` 구조체를 수정하여 텍스처 좌표를 위한 `vec2`를 포함시키세요. 버텍스 셰이더에서 텍스처 좌표에 접근하여 프래그먼트 셰이더로 보낼 수 있도록 `VkVertexInputAttributeDescription`도 추가하는 것이 필요합니다. 이는 표면을 걸쳐 보간하기 위해 필요합니다.

```c++
const std::vector<Vertex> vertices = {
    {{-0.5f, -0.5f}, {1.0f, 0.0f, 0.0f}, {1.0f, 0.0f}},
    {{0.5f, -0.5f}, {0.0f, 1.0f, 0.0f}, {0.0f, 0.0f}},
    {{0.5f, 0.5f}, {0.0f, 0.0f, 1.0f}, {0.0f, 1.0f}},
    {{-0.5f, 0.5f}, {1.0f, 1.0f, 1.0f}, {1.0f, 1.0f}}
};
```

이 튜토리얼에서는 `0, 0` 좌표에서 시작하여 `1, 1` 좌표에서 끝나는 사각형을 텍스처로 채울 것입니다. 다른 좌표를 사용하여 실험해 보세요. `0` 이하나 `1` 이상의 좌표를 사용하여 주소 지정 모드를 확인해 보세요!

## 셰이더

텍스처에서 색상을 샘플링하도록 셰이더를 수정하는 것이 마지막 단계입니다. 먼저 버텍스 셰이더를 수정하여 텍스처 좌표를 프래그먼트 셰이더로 전달해야 합니다:

```glsl
layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;
layout(location = 2) in vec2 inTexCoord;

layout(location = 0) out vec3 fragColor;
layout(location = 1) out vec2 fragTexCoord;

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
    fragTexCoord = inTexCoord;
}
```

프래그먼트 셰이더에서는 다음과 같이 텍스처 좌표를 색상으로 시각화할 수 있습니다:

```glsl
#version 450

layout(location = 0) in vec3 fragColor;
layout(location = 1) in vec2 fragTexCoord;

layout(location = 0) out vec4 outColor;

void main() {
    outColor = vec4(fragTexCoord, 0.0, 1.0);
}
```

아래 이미지와 같이 보여야 합니다. 셰이더를 재컴파일하는 것을 잊지 마세요!

![](/images/texcoord_visualization.png)

결합 이미지 샘플러 디스크립터는 GLSL에서 샘플러 유니폼으로 표현됩니다. 프래그먼트 셰이더에 그것을 참조 추가하세요:

```glsl
layout(binding = 1) uniform sampler2D texSampler;
```

다른 이미지 유형에 대한 `sampler1D` 및 `sampler3D` 유형도 있습니다. 여기서 올바른 바인딩을 사용해야 합니다.

```glsl
void main() {
    outColor = texture(texSampler, fragTexCoord);
}
```

텍스처는 내장된 `texture` 함수를 사용하여 샘플링됩니다. 이 함수는 `sampler`와 좌표를 인수로 받습니다. 샘플러는 배경에서 필터링 및 변환을 자동으로 처리합니다. 이제 애플리케이션을 실행할 때 사각형에 텍스처가 보여야 합니다:

![](/images/texture_on_square.png)

텍스처 좌표를 `1` 이상의 값으로 확장하여 주소 지정 모드를 실험해 보세요. 예를 들어, `VK_SAMPLER_ADDRESS_MODE_REPEAT`을 사용할 때 다음 프래그먼트 셰이더는 아래 이미지에 나타난 결과를 생성합니다:

```glsl
void main() {
    outColor = texture(texSampler, fragTexCoord * 2.0);
}
```

![](/images/texture_on_square_repeated.png)

또한 버텍스 색상을 사용하여 텍스처 색상을 조작할 수 있습니다:

```glsl
void main() {
    outColor = vec4(fragColor * texture(texSampler, fragTexCoord).rgb, 1.0);
}
```

여기서 RGB와 알파 채널을 분리하여 알파 채널을 조정하지 않았습니다.

![](/images/texture_on_square_colorized.png)

셰이더에서 이미지에 접근하는 방법을 이제 알게 되었습니다! 이 기술은 프레임버퍼에도 쓰여지는 이미지와 결합할 때 매우 강력합니다. 이러한 이미지를 입력으로 사용하여 포스트 프로세싱과 카메라 디스플레이와 같은 멋진 효과를 3D 세계 내에서 구현할 수 있습니다.

[C++ 코드](/code/26_texture_mapping.cpp) /
[버텍스 셰이더](/code/26_shader_textures.vert) /
[프래그먼트 셰이더](/code/26_shader_textures.frag)
