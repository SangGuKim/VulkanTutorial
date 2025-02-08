# 컴퓨트 셰이더

## 소개

이번 추가 장에서는 컴퓨트 셰이더에 대해 살펴보겠습니다. 지금까지의 모든 장들은 Vulkan 파이프라인의 전통적인 그래픽 부분에 초점을 맞췄습니다. 그러나 OpenGL과 같은 이전 API와 달리, Vulkan에서는 컴퓨트 셰이더 지원이 필수입니다. 이는 모든 Vulkan 구현에서 컴퓨트 셰이더를 사용할 수 있음을 의미합니다. 고성능 데스크탑 GPU든 저전력 임베디드 디바이스든 상관없이 말이죠.

이로 인해 그래픽 프로세서 유닛(GPU)에서의 일반 목적 컴퓨팅(GPGPU)의 세계가 열렸습니다. GPGPU는 전통적으로 CPU의 영역이었던 일반 계산을 GPU에서 수행할 수 있음을 의미합니다. GPU가 점점 더 강력하고 유연해지면서 CPU의 일반 목적 기능을 요구하는 많은 작업들이 이제 GPU에서 실시간으로 처리될 수 있습니다.

GPU의 컴퓨트 기능을 사용할 수 있는 몇 가지 예는 이미지 조작, 가시성 테스트, 후처리, 고급 조명 계산, 애니메이션, 물리학(예: 입자 시스템) 등이 있습니다. 심지어 그래픽 출력이 필요 없는 계산만을 위한 컴퓨트를 사용하는 것도 가능합니다(예: 숫자 연산 또는 AI 관련 작업). 이를 "헤드리스 컴퓨트"라고 합니다.

## 장점

GPU에서 계산 집약적인 계산을 수행하는 것은 여러 장점이 있습니다. 가장 명확한 장점은 CPU에서 작업을 오프로딩하는 것입니다. 또 다른 장점은 CPU의 주 메모리와 GPU의 메모리 사이에 데이터를 이동할 필요가 없다는 것입니다. 모든 데이터가 GPU에 머무르며 주 메모리에서 느린 전송을 기다릴 필요가 없습니다.

이 외에도, GPU는 수천 개의 작은 컴퓨트 유닛을 갖춘 매우 병렬화된 구조를 가지고 있는 반면, 몇 개의 큰 컴퓨트 유닛을 갖는 CPU보다 고도로 병렬적인 워크플로우에 더 적합할 수 있습니다.

## Vulkan 파이프라인

컴퓨트가 그래픽 파이프라인 부분과 완전히 분리되어 있음을 아는 것이 중요합니다. 공식 사양에서 나온 다음 블록 다이어그램의 Vulkan 파이프라인에서 이를 볼 수 있습니다:

![](/images/vulkan_pipeline_block_diagram.png)

이 다이어그램에서 왼쪽에는 전통적인 그래픽 파이프라인 부분을 볼 수 있고, 오른쪽에는 이 그래픽 파이프라인의 일부가 아닌 여러 단계를 볼 수 있습니다. 컴퓨트 셰이더(단계)와 같은 것들이죠. 컴퓨트 셰이더 단계가 그래픽 파이프라인과 분리되어 있기 때문에 필요한 곳에서 언제든지 사용할 수 있습니다. 예를 들어, 프래그먼트 셰이더는 항상 버텍스 셰이더의 변환된 출력에 적용되는 반면 말이죠.

다이어그램의 중앙에서도 디스크립터 세트 등을 컴퓨트에서도 사용한다는 것을 알 수 있습니다. 따라서 디스크립터 레이아웃, 디스크립터 세트 및 디스크립터에 대해 배운 모든 것이 여기에도 적용됩니다.

## 예제

이 장에서 구현할 이해하기 쉬운 예제는 GPU 기반 입자 시스템입니다. 이러한 시스템은 많은 게임에서 사용되며, 대개 수천 개의 입자가 상호 작용하는 프레임 속도로 업데이트되어야 합니다. 이러한 시스템을 렌더링하는 데는 두 가지 주요 구성 요소가 필요합니다: 버텍스 버퍼로 전달된 정점과 어떤 방정식에 기반하여 이들을 업데이트하는 방법입니다.

"전통적인" CPU 기반 입자 시스템은 입자 데이터를 시스템의 주 메모리에 저장하고 CPU를 사용하여 이를 업데이트합니다. 업데이트 후, 정점을 GPU의 메모리로 다시 전송하여 다음 프레임에서 업데이트된 입자를 표시할 수 있습니다. 가장 직관적인 방법은 각 프레임마다 정점 버퍼를 새 데이터로 다시 생성하는 것입니다. 이는 분명히 매우 비용이 많이 듭니다. 구현에 따라 다른 옵션들도 있습니다. 예를 들어 데스크탑 시스템에서는 "resizable BAR"로 알려진 GPU 메모리 매핑을 사용하거나, 통합 GPU에서 통합 메모리를 사용하는 것입니다. 또는 호스트 로컬 버퍼를 사용하는 방법도 있습니다(이는 PCI-E 대역폭 때문에 가장 느린 방법일 것입니다). 하지만 어떤 버퍼 업데이트 방법을 선택하든, 입자를 업데이트하기 위해 항상 "왕복" CPU가 필요합니다.

GPU 기반 입자 시스템에서는 이러한 왕복이 더 이상 필요하지 않습니다. 정점은 시작할 때 한 번만 GPU로 업로드되고 모든 업데이트는 GPU의 메모리에서 컴퓨트 셰이더를 사용하여 수행됩니다. 이것이 더 빠른 주된 이유 중 하나는 GPU와 로컬 메모리 간의 훨씬 높은 대역폭 때문입니다. CPU 기반 시나리오에서는 주 메모리 및 PCI-익스프레스 대역폭에 의해 제한될 것이며, 이는 종종 GPU의 메모리 대역폭의 일부에 불과합니다.

GPU에 전용 컴퓨트 큐가 있는 경우, 그래픽 파이프라인의 렌더링 부분과 병렬로 입자를 업데이트할 수 있습니다. 이를 "비동기 컴퓨트"라고 하며, 이 튜토리얼에서 다루지 않는 고급 주제입니다.

이 장의 코드에서 캡처된 스크린샷은 다음과 같습니다. 여기에 표시된 입자들은 CPU의 개입 없이 GPU에서 직접 컴퓨트 셰이더에 의해 업데이트됩니다:

![](/images/compute_shader_particles.png)

## 데이터 조작

이 튜토리얼에서 우리는 이미 정점 및 인덱스 버퍼를 통해 기본 요소를 전달하고 유니폼 버퍼를 통해 셰이더에 데이터를 전달하는 다양한 버퍼 유형에 대해 배웠습니다. 우리는 또한 텍스처 매핑을 수행하기 위해 이미지를 사용했습니다. 그러나 지금까지 우리는 항상 CPU를 사용하여 데이터를 작성하고 GPU에서만 읽기를 수행했습니다.

컴퓨트 셰이더와 함께 도입된 중요한 개념은 버퍼에서 임의로 읽기 **및 쓰기**를 수행할 수 있다는 것입니다. 이를 위해 Vulkan은 두 가지 전용 저장 유형을 제공합니다.

### 셰이더 저장 버퍼 객체 (SSBO)

셰이더 저장 버퍼(SSBO)는 셰이더가 버퍼에서 읽고 쓸 수 있게 합니다. 유니폼 버퍼 객체 사용과 비슷하지만 SSBO를 다른 버퍼 유형에 별칭으로 사용할 수 있고, 임의의 크기로 만들 수 있다는 점이 가장 큰 차이입니다.

GPU 기반 입자 시스템에 대해 다시 생각해 보면, 컴퓨트 셰이더가 업데이트(쓰기)하고 버텍스 셰이더가 읽기(그리기)하는 정점을 다루는 방법이 궁금할 수 있습니다.

하지만 이는 문제가 되지 않습니다. Vulkan에서는 버퍼와 이미지에 여러 용도를 지정할 수 있습니다. 그래서 입자 정점 버퍼를 그래픽 패스에서는 정점 버퍼로, 컴퓨트 패스에서는 저장 버퍼로 사용하려면, 이 두 가지 사용 플래그를 포함하여 버퍼를 생성하기만 하면 됩니다:

```cpp
VkBufferCreateInfo bufferInfo{};
...
bufferInfo.usage = VK_BUFFER_USAGE_VERTEX_BUFFER_BIT | VK_BUFFER_USAGE_STORAGE_BUFFER_BIT | VK_BUFFER_USAGE_TRANSFER_DST_BIT;
...

if (vkCreateBuffer(device, &bufferInfo, nullptr, &shaderStorageBuffers[i]) != VK_SUCCESS) {
    throw std::runtime_error("failed to create vertex buffer!");
}
```

`VK_BUFFER_USAGE_VERTEX_BUFFER_BIT`와 `VK_BUFFER_USAGE_STORAGE_BUFFER_BIT` 플래그를 `bufferInfo.usage`에 설정하여 이 버퍼를 두 가지 시나리오에 사용하려고 한다는 것을 구현에 알려줍니다. 여기에 `VK_BUFFER_USAGE_TRANSFER_DST_BIT` 플래그도 추가해서 호스트에서 GPU로 데이터를 전송할 수 있습니다. 이는 셰이더 저장 버퍼를 GPU 메모리에만 두고 싶기 때문에 필수적입니다(`VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT`).

다음은 `createBuffer` 도우미 함수를 사용한 동일한 코드입니다:

```cpp
createBuffer(bufferSize, VK_BUFFER_USAGE_STORAGE_BUFFER_BIT | VK_BUFFER_USAGE_VERTEX_BUFFER_BIT | VK_BUFFER_USAGE_TRANSFER_DST_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, shaderStorageBuffers[i], shaderStorageBuffersMemory[i]);
```

이와 같은 버퍼에 접근하는 GLSL 셰이더 선언은 다음과 같습니다:

```glsl
struct Particle {
  vec2 position;
  vec2 velocity;
  vec4 color;
};

layout(std140, binding = 1) readonly buffer ParticleSSBOIn {
   Particle particlesIn[ ];
};

layout(std140, binding = 2) buffer ParticleSSBOOut {
   Particle particlesOut[ ];
};
```

이 예제에서는 각 입자가 위치와 속도 값을 갖는 타입화된 SSBO를 가지고 있습니다(`Particle` 구조체 참조). SSBO는 `[]`로 표시된 것처럼 무제한 수의 입자를 포함합니다. SSBO에서 요소 수를 지정할 필요가 없다는 것은 유니폼 버퍼에 비해 장점 중 하나입니다. `std140`은 셰이더 저장 버퍼의 구성원 요소가 메모리에 어떻게 정렬되는지 결정하는 메모리 레이아웃 한정자입니다. 이는 호스트와 GPU 간에 버퍼를 매핑할 때 필요한 특정 보장을 제공합니다.

컴퓨트 셰이더에서 이러한 저장 버퍼 객체에 쓰는 것은 C++ 측에서 버퍼에 쓰는 것과 비슷하고 간단합니다:

```glsl
particlesOut[index].position = particlesIn[index].position + particlesIn[index].velocity.xy * ubo.deltaTime;
```

### 저장 이미지

*참고: 이 장에서는 이미지 조작을 수행하지 않습니다. 이 문단은 컴퓨트 셰이더를 사용하여 이미지 조작도 가능하다는 것을 독자에게 알리기 위한 것입니다.*

저장 이미지는 이미지를 읽고 쓸 수 있게 합니다. 일반적인 사용 사례는 텍스처에 이미지 효과를 적용하거나, 후처리를 수행하거나(매우 비슷한 작업), 미합맵을 생성하는 것입니다.

이미지에 대해서도 비슷합니다:

```cpp
VkImageCreateInfo imageInfo {};
...
imageInfo.usage = VK_IMAGE_USAGE_SAMPLED_BIT | VK_IMAGE_USAGE_STORAGE_BIT;
...

if (vkCreateImage(device, &imageInfo, nullptr, &textureImage) != VK_SUCCESS) {
    throw std::runtime_error("failed to create image!");
}
```

`VK_IMAGE_USAGE_SAMPLED_BIT` 및 `VK_IMAGE_USAGE_STORAGE_BIT` 플래그는 구현에 이 이미지를 두 가지 시나리오에 사용하려고 한다는 것을 알려줍니다: 프래그먼트 셰이더에서 샘플링된 이미지로, 컴퓨터 셰이더에서 저장 이미지로 사용됩니다;

저장 이미지를 선언하는 GLSL 셰이더 선언은 프래그먼트 셰이더에서 사용되는 샘플링된 이미지와 비슷합니다:

```glsl
layout (binding = 0, rgba8) uniform readonly image2D inputImage;
layout (binding = 1, rgba8) uniform writeonly image2D outputImage;
```

여기에서 몇 가지 차이점은 이미지 형식을 위한 추가 속성인 `rgba8`, 구현에 우리가 입력 이미지에서만 읽고 출력 이미지에만 쓸 것임을 알리는 `readonly` 및 `writeonly` 한정자, 그리고 저장 이미지를 선언하기 위해 `image2D` 유형을 사용해야 합니다.

컴퓨트 셰이더에서 저장 이미지를 읽고 쓰는 것은 `imageLoad` 및 `imageStore`를 사용하여 수행됩니다:

```glsl
vec3 pixel = imageLoad(inputImage, ivec2(gl_GlobalInvocationID.xy)).rgb;
imageStore(outputImage, ivec2(gl_GlobalInvocationID.xy), pixel);
```

### 컴퓨트 큐 패밀리

[물리적 장치 및 큐 패밀리 장](03_Drawing_a_triangle/00_Setup/03_Physical_devices_and_queue_families.md#page_Queue-families)에서 이미 큐 패밀리에 대해 배웠고, 그래픽 큐 패밀리를 선택하는 방법을 배웠습니다. 컴퓨트는 큐 패밀리 속성 플래그 비트 `VK_QUEUE_COMPUTE_BIT`를 사용합니다. 그래서 컴퓨트 작업을 수행하려면 컴퓨트를 지원하는 큐 패밀리에서 큐를 가져와야 합니다.

Vulkan은 그래픽 연산을 지원하는 구현이 적어도 하나의 큐 패밀리를 가지고 있어야 하며, 이는 그래픽 및 컴퓨트 연산을 모두 지원해야 합니다. 하지만 구현에 따라 전용 컴퓨트 큐를 제공할 수도 있습니다. 이 전용 컴퓨트 큐(그래픽 비트가 없는)는 비동기 컴퓨트 큐를 암시합니다. 그러나 이 튜토리얼은 초보자 친화적이므로 그래픽 및 컴퓨트 연산을 모

두 수행할 수 있는 큐를 사용할 것입니다. 이는 여러 고급 동기화 메커니즘을 다루지 않아도 되므로 더 간단합니다.

컴퓨트 샘플을 위해 장치 생성 코드를 조금 변경해야 합니다:

```cpp
uint32_t queueFamilyCount = 0;
vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, nullptr);

std::vector<VkQueueFamilyProperties> queueFamilies(queueFamilyCount);
vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, queueFamilies.data());

int i = 0;
for (const auto& queueFamily : queueFamilies) {
    if ((queueFamily.queueFlags & VK_QUEUE_GRAPHICS_BIT) && (queueFamily.queueFlags & VK_QUEUE_COMPUTE_BIT)) {
        indices.graphicsAndComputeFamily = i;
    }

    i++;
}
```

변경된 큐 패밀리 인덱스 선택 코드는 이제 그래픽 및 컴퓨트를 모두 지원하는 큐 패밀리를 찾으려고 시도할 것입니다.

그런 다음 `createLogicalDevice`에서 이 큐 패밀리에서 컴퓨트 큐를 가져올 수 있습니다:

```cpp
vkGetDeviceQueue(device, indices.graphicsAndComputeFamily.value(), 0, &computeQueue);
```

## 컴퓨트 셰이더 단계

그래픽 샘플에서 우리는 서로 다른 파이프라인 단계에서 셰이더를 로드하고 디스크립터에 액세스했습니다. 컴퓨트 셰이더는 `VK_SHADER_STAGE_COMPUTE_BIT` 파이프라인을 사용하여 유사한 방식으로 액세스됩니다. 따라서 컴퓨트 셰이더를 로드하는 것은 버텍스 셰이더를 로드하는 것과 동일하지만 다른 셰이더 단계를 사용합니다. 다음 단락에서 이에 대해 자세히 설명할 것입니다. 컴퓨트는 또한 나중에 사용할 디스크립터 및 파이프라인에 대한 새로운 바인딩 지점 유형인 `VK_PIPELINE_BIND_POINT_COMPUTE`를 도입합니다.

## 컴퓨트 셰이더 로드

애플리케이션에서 컴퓨트 셰이더를 로드하는 것은 다른 셰이더를 로드하는 것과 같습니다. 유일한 실제 차이점은 위에서 언급한 `VK_SHADER_STAGE_COMPUTE_BIT`를 사용해야 한다는 것입니다.

```cpp
auto computeShaderCode = readFile("shaders/compute.spv");

VkShaderModule computeShaderModule = createShaderModule(computeShaderCode);

VkPipelineShaderStageCreateInfo computeShaderStageInfo{};
computeShaderStageInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
computeShaderStageInfo.stage = VK_SHADER_STAGE_COMPUTE_BIT;
computeShaderStageInfo.module = computeShaderModule;
computeShaderStageInfo.pName = "main";
...
```

## 셰이더 저장 버퍼 준비

이전에 배웠듯이, 셰이더 저장 버퍼를 사용하여 컴퓨트 셰이더에 임의의 데이터를 전달할 수 있습니다. 이 예제에서는 GPU에 입자 배열을 업로드하여 GPU의 메모리에서 직접 조작할 수 있습니다.

[프레임 인 플라이트](03_Drawing_a_triangle/03_Drawing/03_Frames_in_flight.md) 장에서 프레임 인 플라이트 당 리소스를 중복하여 CPU와 GPU를 계속 작업할 수 있도록 했습니다. 먼저 버퍼 객체와 이를 백업하는 디바이스 메모리에 대한 벡터를 선언합니다:

```cpp
std::vector<VkBuffer> shaderStorageBuffers;
std::vector<VkDeviceMemory> shaderStorageBuffersMemory;
```

`createShaderStorageBuffers`에서는 이 벡터들을 최대 프레임 수 인 플라이트와 일치하도록 크기를 조정합니다:

```cpp
shaderStorageBuffers.resize(MAX_FRAMES_IN_FLIGHT);
shaderStorageBuffersMemory.resize(MAX_FRAMES_IN_FLIGHT);
```

이 설정이 완료되면 호스트 측에서 입자 정보를 초기화하여 GPU로 이동을 시작할 수 있습니다:

```cpp
    // 입자 초기화
    std::default_random_engine rndEngine((unsigned)time(nullptr));
    std::uniform_real_distribution<float> rndDist(0.0f, 1.0f);

    // 원형 위의 초기 입자 위치
    std::vector<Particle> particles(PARTICLE_COUNT);
    for (auto& particle : particles) {
        float r = 0.25f * sqrt(rndDist(rndEngine));
        float theta = rndDist(rndEngine) * 2 * 3.14159265358979323846;
        float x = r * cos(theta) * HEIGHT / WIDTH;
        float y = r * sin(theta);
        particle.position = glm::vec2(x, y);
        particle.velocity = glm::normalize(glm::vec2(x,y)) * 0.00025f;
        particle.color = glm::vec4(rndDist(rndEngine), rndDist(rndEngine), rndDist(rndEngine), 1.0f);
    }

```

그런 다음 호스트의 메모리에 [스테이징 버퍼](04_Vertex_buffers/02_Staging_buffer.md)를 생성하여 초기 입자 속성을 보관합니다:

```cpp
    VkDeviceSize bufferSize = sizeof(Particle) * PARTICLE_COUNT;

    VkBuffer stagingBuffer;
    VkDeviceMemory stagingBufferMemory;
    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, stagingBuffer, stagingBufferMemory);

    void* data;
    vkMapMemory(device, stagingBufferMemory, 0, bufferSize, 0, &data);
    memcpy(data, particles.data(), (size_t)bufferSize);
    vkUnmapMemory(device, stagingBufferMemory);
```    

이 스테이징 버퍼를 소스로 사용하여 프레임당 셰이더 저장 버퍼를 생성하고 스테이징 버퍼에서 각각의 셰이더 저장 버퍼로 입자 속성을 복사합니다:

```cpp
    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        createBuffer(bufferSize, VK_BUFFER_USAGE_STORAGE_BUFFER_BIT | VK_BUFFER_USAGE_VERTEX_BUFFER_BIT | VK_BUFFER_USAGE_TRANSFER_DST_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, shaderStorageBuffers[i], shaderStorageBuffersMemory[i]);
        // 스테이징 버퍼(호스트)에서 셰이더 저장 버퍼(GPU)로 데이터 복사
        copyBuffer(stagingBuffer, shaderStorageBuffers[i], bufferSize);
    }
}
```

## 디스크립터

컴퓨트를 위한 디스크립터 설정은 그래픽과 거의 동일합니다. 유일한 차이점은 디스크립터가 컴퓨트 단계에서 접근 가능하도록 `VK_SHADER_STAGE_COMPUTE_BIT`를 설정해야 한다는 것입니다:

```cpp
std::array<VkDescriptorSetLayoutBinding, 3> layoutBindings{};
layoutBindings[0].binding = 0;
layoutBindings[0].descriptorCount = 1;
layoutBindings[0].descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
layoutBindings[0].pImmutableSamplers = nullptr;
layoutBindings[0].stageFlags = VK_SHADER_STAGE_COMPUTE_BIT;
...
```

여기에서 셰이더 단계를 결합할 수 있으므로, 디스크립터가 버텍스 및 컴퓨트 단계에서 접근 가능하게 하려면, 두 단계의 비트를 모두 설정하면 됩니다:

```cpp
layoutBindings[0].stageFlags = VK_SHADER_STAGE_VERTEX_BIT | VK_SHADER_STAGE_COMPUTE_BIT;
```

샘플에 대한 디스크립터 설정은 다음과 같습니다

. 레이아웃은 다음과 같습니다:

```cpp
std::array<VkDescriptorSetLayoutBinding, 3> layoutBindings{};
layoutBindings[0].binding = 0;
layoutBindings[0].descriptorCount = 1;
layoutBindings[0].descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
layoutBindings[0].pImmutableSamplers = nullptr;
layoutBindings[0].stageFlags = VK_SHADER_STAGE_COMPUTE_BIT;

layoutBindings[1].binding = 1;
layoutBindings[1].descriptorCount = 1;
layoutBindings[1].descriptorType = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER;
layoutBindings[1].pImmutableSamplers = nullptr;
layoutBindings[1].stageFlags = VK_SHADER_STAGE_COMPUTE_BIT;

layoutBindings[2].binding = 2;
layoutBindings[2].descriptorCount = 1;
layoutBindings[2].descriptorType = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER;
layoutBindings[2].pImmutableSamplers = nullptr;
layoutBindings[2].stageFlags = VK_SHADER_STAGE_COMPUTE_BIT;

VkDescriptorSetLayoutCreateInfo layoutInfo{};
layoutInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO;
layoutInfo.bindingCount = 3;
layoutInfo.pBindings = layoutBindings.data();

if (vkCreateDescriptorSetLayout(device, &layoutInfo, nullptr, &computeDescriptorSetLayout) != VK_SUCCESS) {
    throw std::runtime_error("failed to create compute descriptor set layout!");
}
```

이 설정을 보면 하나의 입자 시스템만 렌더링하는데도 불구하고 셰이더 저장 버퍼 객체에 대한 두 개의 레이아웃 바인딩이 있는 이유가 궁금할 수 있습니다. 이는 입자 위치가 델타 시간에 기반하여 프레임마다 업데이트되기 때문입니다. 즉, 각 프레임은 지난 프레임의 입자 위치를 알아야 하므로 새 델타 시간으로 업데이트하고 자신의 SSBO에 쓸 수 있습니다:

![](/images/compute_ssbo_read_write.svg)

이를 위해 컴퓨트 셰이더에서 지난 프레임과 현재 프레임의 SSBO에 모두 접근할 수 있도록 디스크립터 설정에서 두 SSBO를 모두 컴퓨트 셰이더에 전달합니다. `storageBufferInfoLastFrame`과 `storageBufferInfoCurrentFrame`를 참조하세요:

```cpp
for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
    VkDescriptorBufferInfo uniformBufferInfo{};
    uniformBufferInfo.buffer = uniformBuffers[i];
    uniformBufferInfo.offset = 0;
    uniformBufferInfo.range = sizeof(UniformBufferObject);

    std::array<VkWriteDescriptorSet, 3> descriptorWrites{};
    ...

    VkDescriptorBufferInfo storageBufferInfoLastFrame{};
    storageBufferInfoLastFrame.buffer = shaderStorageBuffers[(i - 1) % MAX_FRAMES_IN_FLIGHT];
    storageBufferInfoLastFrame.offset = 0;
    storageBufferInfoLastFrame.range = sizeof(Particle) * PARTICLE_COUNT;

    descriptorWrites[1].sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
    descriptorWrites[1].dstSet = computeDescriptorSets[i];
    descriptorWrites[1].dstBinding = 1;
    descriptorWrites[1].dstArrayElement = 0;
    descriptorWrites[1].descriptorType = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER;
    descriptorWrites[1].descriptorCount = 1;
    descriptorWrites[1].pBufferInfo = &storageBufferInfoLastFrame;

    VkDescriptorBufferInfo storageBufferInfoCurrentFrame{};
    storageBufferInfoCurrentFrame.buffer = shaderStorageBuffers[i];
    storageBufferInfoCurrentFrame.offset = 0;
    storageBufferInfoCurrentFrame.range = sizeof(Particle) * PARTICLE_COUNT;

    descriptorWrites[2].sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
    descriptorWrites[2].dstSet = computeDescriptorSets[i];
    descriptorWrites[2].dstBinding = 2;
    descriptorWrites[2].dstArrayElement = 0;
    descriptorWrites[2].descriptorType = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER;
    descriptorWrites[2].descriptorCount = 1;
    descriptorWrites[2].pBufferInfo = &storageBufferInfoCurrentFrame;

    vkUpdateDescriptorSets(device, 3, descriptorWrites.data(), 0, nullptr);
}
```

SSBO에 대한 디스크립터 유형을 디스크립터 풀에서 요청해야 함을 기억하세요:

```cpp
std::array<VkDescriptorPoolSize, 2> poolSizes{};
...

poolSizes[1].type = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER;
poolSizes[1].descriptorCount = static_cast<uint32_t>(MAX_FRAMES_IN_FLIGHT) * 2;
```

세트에서 지난 프레임과 현재 프레임의 SSBO를 참조하기 때문에 풀에서 요청하는 `VK_DESCRIPTOR_TYPE_STORAGE_BUFFER` 유형의 수를 두 배로 늘려야 합니다.

## 컴퓨트 파이프라인

컴퓨트는 그래픽 파이프라인의 일부가 아니므로 `vkCreateGraphicsPipelines`를 사용할 수 없습니다. 대신 `vkCreateComputePipelines`를 사용하여 컴퓨트 명령을 실행하기 위한 전용 컴퓨트 파이프라인을 생성해야 합니다. 컴퓨트 파이프라인은 래스터화 상태를 전혀 건드리지 않으므로 그래픽 파이프라인보다 상태가 훨씬 적습니다:

```cpp
VkComputePipelineCreateInfo pipelineInfo{};
pipelineInfo.sType = VK_STRUCTURE_TYPE_COMPUTE_PIPELINE_CREATE_INFO;
pipelineInfo.layout = computePipelineLayout;
pipelineInfo.stage = computeShaderStageInfo;

if (vkCreateComputePipelines(device, VK_NULL_HANDLE, 1, &pipelineInfo, nullptr, &computePipeline) != VK_SUCCESS) {
    throw std::runtime_error("failed to create compute pipeline!");
}
```

설정은 훨씬 간단하며, 하나의 셰이더 단계와 파이프라인 레이아웃만 필요합니다. 그래픽 파이프라인과 마찬가지로 파이프라인 레이아웃이 작동합니다:

```cpp
VkPipelineLayoutCreateInfo pipelineLayoutInfo{};
pipelineLayoutInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
pipelineLayoutInfo.setLayoutCount = 1;
pipelineLayoutInfo.pSetLayouts = &computeDescriptorSetLayout;

if (vkCreatePipelineLayout(device, &pipelineLayoutInfo, nullptr, &computePipelineLayout) != VK_SUCCESS) {
    throw std::runtime_error("failed to create compute pipeline layout!");
}
```

## 컴퓨트 공간

컴퓨트 셰이더의 작동 방식과 GPU에 컴퓨트 작업을 제출하는 방법에 대해 이야기하기 전에, 컴퓨트 작업이 GPU의 컴퓨트 하드웨어에 의해 어떻게 처리되는지 정의하는 두 가지 중요한 컴퓨트 개념인 **워크 그룹**과 **호출**에 대해 이야기해야 합니다. 그들은 세 차원(x, y, z)에서 컴퓨트 작업이 처리되는 추상 실행 모델을 정의합니다.

**워크 그룹**은 컴퓨트 작업이 GPU의 컴퓨트 하드웨어에 의해 어떻게 형성되고 처리되는지를 정의합니다. GPU가 작업해야 할 작업 항목으로 생각할 수 있습니다. 워크 그룹 차원은 명령 버퍼 시간에 애플리케이션에 의해 설정됩니다.

그리고 각 워크 그룹은 동일한 컴퓨트 셰이더를 실행하는 **호출**의 모음입니다. 호출은 잠재적으로 병렬로 실행될 수 있으며 그 차원은 컴퓨트 셰이더에서 설정됩니다. 단일 워크그룹 내의 호출은 공유 메모리에 접근할 수 있습니다.

이 이미지는 세 차원에서 이 두 가지의 관계를 보여줍니다:

![](/images/compute_space.svg)

워

크 그룹(정의된 `vkCmdDispatch`에 의해)과 호출(컴퓨트 셰이더에서 로컬 크기로 정의된)의 차원 수는 입력 데이터가 어떻게 구성되어 있는지에 따라 다릅니다. 예를 들어, 1차원 배열에서 작업하는 경우 x 차원만 지정해야 합니다.

예를 들어: 워크 그룹 수[64, 1, 1]와 컴퓨트 셰이더 로컬 크기[32, 32, 1]로 디스패치를 수행하면 컴퓨트 셰이더가 64 x 32 x 32 = 65,536번 호출됩니다.

워크 그룹 수와 로컬 크기의 최대 카운트는 구현마다 다르므로 항상 `VkPhysicalDeviceLimits`의 `maxComputeWorkGroupCount`, `maxComputeWorkGroupInvocations` 및 `maxComputeWorkGroupSize`와 같은 컴퓨트 관련 제한을 확인해야 합니다.

## 컴퓨트 셰이더

이제 컴퓨트 셰이더 파이프라인을 설정하는 데 필요한 모든 부분에 대해 배웠으므로 컴퓨트 셰이더에 대해 살펴볼 시간입니다. 버텍스 및 프래그먼트 셰이더 등에서 GLSL 셰이더를 사용하는 것과 관련된 모든 것들이 컴퓨트 셰이더에도 적용됩니다. 문법은 동일하며 애플리케이션과 셰이더 간에 데이터를 전달하는 많은 개념이 동일합니다. 그러나 몇 가지 중요한 차이점이 있습니다.

선형 배열의 입자를 업데이트하기 위한 매우 기본적인 컴퓨트 셰이더는 다음과 같을 수 있습니다:

```glsl
#version 450

layout (binding = 0) uniform ParameterUBO {
    float deltaTime;
} ubo;

struct Particle {
    vec2 position;
    vec2 velocity;
    vec4 color;
};

layout(std140, binding = 1) readonly buffer ParticleSSBOIn {
   Particle particlesIn[ ];
};

layout(std140, binding = 2) buffer ParticleSSBOOut {
   Particle particlesOut[ ];
};

layout (local_size_x = 256, local_size_y = 1, local_size_z = 1) in;

void main() 
{
    uint index = gl_GlobalInvocationID.x;  

    Particle particleIn = particlesIn[index];

    particlesOut[index].position = particleIn.position + particleIn.velocity.xy * ubo.deltaTime;
    particlesOut[index].velocity = particleIn.velocity;
    ...
}
```

셰이더의 맨 위 부분은 셰이더 입력에 대한 선언을 포함합니다. 첫 번째는 바인딩 0에서 유니폼 버퍼 객체입니다. 이미 이 튜토리얼에서 배운 것입니다. 그 아래에는 C++ 코드에서 선언과 일치하는 Particle 구조체를 선언합니다. 바인딩 1은 지난 프레임의 입자 데이터가 있는 셰이더 저장 버퍼 객체(SSBO)를 참조하며, 바인딩 2는 이 셰이더가 업데이트할 현재 프레임의 SSBO를 가리킵니다.

흥미로운 것은 이 컴퓨트 전용 선언과 관련된 것입니다:

```glsl
layout (local_size_x = 256, local_size_y = 1, local_size_z = 1) in;
```
이것은 현재 워크 그룹에서 이 컴퓨트 셰이더의 호출 수를 정의합니다. 앞서 언급했듯이, 이것은 컴퓨트 공간의 로컬 부분입니다. 따라서 `local_` 접두사가 붙습니다. 우리가 1D 입자 배열에서 작업하기 때문에 `local_size_x`의 x 차원에 대한 숫자만 지정할 필요가 있습니다.

`main` 함수는 그런 다음 지난 프레임의 SSBO에서 읽고 현재 프레임의 SSBO에 업데이트된 입자 위치를 씁니다. 다른 셰이더 유형과 마찬가지로 컴퓨트 셰이더에는 자체적인 내장 입력 변수 세트가 있습니다. 내장된 것들은 항상 `gl_` 접두사로 시작합니다. 그 중 하나는 현재 디스패치에서 전체적으로 현재 컴퓨트 셰이더 호출을 고유하게 식별하는 변수인 `gl_GlobalInvocationID`입니다. 우리는 이것을 입자 배열에 인덱스로 사용합니다.

## 컴퓨트 명령 실행

### 디스패치

이제 GPU에 실제로 컴퓨트 작업을 지시할 시간입니다. 이는 명령 버퍼 내에서 `vkCmdDispatch`를 호출하여 수행됩니다. 완벽하게 사실은 아니지만, 디스패치는 컴퓨트에 대한 드로우 콜인 `vkCmdDraw`와 같습니다. 이 디스패치는 최대 세 차원에서 주어진 수의 컴퓨트 작업 항목을 디스패치합니다.

```cpp
VkCommandBufferBeginInfo beginInfo{};
beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;

if (vkBeginCommandBuffer(commandBuffer, &beginInfo) != VK_SUCCESS) {
    throw std::runtime_error("failed to begin recording command buffer!");
}

...

vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_COMPUTE, computePipeline);
vkCmdBindDescriptorSets(commandBuffer, VK_PIPELINE_BIND_POINT_COMPUTE, computePipelineLayout, 0, 1, &computeDescriptorSets[i], 0, 0);

vkCmdDispatch(computeCommandBuffer, PARTICLE_COUNT / 256, 1, 1);

...

if (vkEndCommandBuffer(commandBuffer) != VK_SUCCESS) {
    throw std::runtime_error("failed to record command buffer!");
}
```

`vkCmdDispatch`는 x 차원에서 `PARTICLE_COUNT / 256`의 로컬 워크 그룹을 디스패치합니다. 우리의 입자 배열이 선형이기 때문에 다른 두 차원은 하나로 두어 일차원 디스패치가 됩니다. 그러나 왜 입자 수(배열 내)를 256으로 나누는지 궁금할 수 있습니다. 그 이유는 이전 단락에서 우리가 설정한 것처럼 각 컴퓨트 셰이더 워크 그룹이 256개의 호출을 수행하기 때문입니다. 따라서 4096개의 입자가 있다면 16개의 워크 그룹을 디스패치하며, 각 워크 그룹은 256개의 컴퓨트 셰이더 호출을 실행합니다. 두 숫자를 올바르게 얻는 것은 일반적으로 작업 부하와 실행 중인 하드웨어에 따라 조정하고 프로파일링하는 데 시간이 걸립니다. 입자 크기가 동적이고 예를 들어 256으로 항상 나눌 수 없는 경우, 컴퓨트 셰이더의 시작 부분에서 `gl_GlobalInvocationID`를 사용하여 전역 호출 인덱스가 입자 수보다 클 경우 반환할 수 있습니다.

컴퓨트 파이프라인과 마찬가지로 컴퓨트 명령 버퍼는 그래픽

 명령 버퍼보다 상태가 훨씬 적습니다. 렌더 패스를 시작하거나 뷰포트를 설정할 필요가 없습니다.

### 작업 제출

우리의 샘플이 컴퓨트와 그래픽 연산을 모두 수행하기 때문에, 우리는 프레임마다 그래픽 및 컴퓨트 큐에 두 번 제출할 것입니다( `drawFrame` 함수 참조):

```cpp
...
if (vkQueueSubmit(computeQueue, 1, &submitInfo, nullptr) != VK_SUCCESS) {
    throw std::runtime_error("failed to submit compute command buffer!");
};
...
if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, inFlightFences[currentFrame]) != VK_SUCCESS) {
    throw std::runtime_error("failed to submit draw command buffer!");
}
```

첫 번째 제출은 컴퓨트 셰이더를 사용하여 입자 위치를 업데이트하고, 두 번째 제출은 그 업데이트된 데이터를 사용하여 입자 시스템을 그립니다.

### 그래픽과 컴퓨트 동기화

Vulkan에서 동기화는 매우 중요한 부분이며, 특히 컴퓨트와 그래픽을 함께 수행할 때 더욱 그렇습니다. 잘못되거나 부족한 동기화는 컴퓨트 셰이더가 업데이트(=쓰기)를 마치지 않은 상태에서 버텍스 단계가 입자를 그리기 시작(=읽기)하는 것(read-after-write 위험) 또는 컴퓨트 셰이더가 아직 파이프라인의 버텍스 부분에서 사용 중인 입자를 업데이트하기 시작할 수 있습니다( write-after-read 위험).

따라서 이러한 경우가 발생하지 않도록 올바르게 동기화해야 합니다. 컴퓨트 작업을 제출하는 방법에 따라 다양한 방법이 있지만, 우리의 경우 두 개의 별도 제출로 처리하므로, 그래픽과 컴퓨트 하드를 동기화하려면 [세마포어](03_Drawing_a_triangle/03_Drawing/02_Rendering_and_presentation.md#page_Semaphores)와 [펜스](03_Drawing_a_triangle/03_Drawing/02_Rendering_and_presentation.md#page_Fences)를 사용합니다.

`createSyncObjects`에서 새로운 컴퓨트 작업 동기화 기본 설정을 추가합니다. 컴퓨트 펜스는 그래픽 펜스와 마찬가지로 신호된 상태에서 생성됩니다. 그렇지 않으면 첫 번째 드로우가 펜스가 신호될 때까지 시간 초과가 발생할 수 있기 때문입니다([여기](03_Drawing_a_triangle/03_Drawing/02_Rendering_and_presentation.md#page_Waiting-for-the-previous-frame)에 자세히 설명되어 있습니다):

```cpp
std::vector<VkFence> computeInFlightFences;
std::vector<VkSemaphore> computeFinishedSemaphores;
...
computeInFlightFences.resize(MAX_FRAMES_IN_FLIGHT);
computeFinishedSemaphores.resize(MAX_FRAMES_IN_FLIGHT);

VkSemaphoreCreateInfo semaphoreInfo{};
semaphoreInfo.sType = VK_STRUCTURE_TYPE_SEMAPHORE_CREATE_INFO;

VkFenceCreateInfo fenceInfo{};
fenceInfo.sType = VK_STRUCTURE_TYPE_FENCE_CREATE_INFO;
fenceInfo.flags = VK_FENCE_CREATE_SIGNALED_BIT;

for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
    ...
    if (vkCreateSemaphore(device, &semaphoreInfo, nullptr, &computeFinishedSemaphores[i]) != VK_SUCCESS ||
        vkCreateFence(device, &fenceInfo, nullptr, &computeInFlightFences[i]) != VK_SUCCESS) {
        throw std::runtime_error("failed to create compute synchronization objects for a frame!");
    }
}
```
그런 다음 이를 사용하여 컴퓨트 버퍼 제출과 그래픽 제출을 동기화합니다:

```cpp
// 컴퓨트 제출
vkWaitForFences(device, 1, &computeInFlightFences[currentFrame], VK_TRUE, UINT64_MAX);

updateUniformBuffer(currentFrame);

vkResetFences(device, 1, &computeInFlightFences[currentFrame]);

vkResetCommandBuffer(computeCommandBuffers[currentFrame], /*VkCommandBufferResetFlagBits*/ 0);
recordComputeCommandBuffer(computeCommandBuffers[currentFrame]);

submitInfo.commandBufferCount = 1;
submitInfo.pCommandBuffers = &computeCommandBuffers[currentFrame];
submitInfo.signalSemaphoreCount = 1;
submitInfo.pSignalSemaphores = &computeFinishedSemaphores[currentFrame];

if (vkQueueSubmit(computeQueue, 1, &submitInfo, computeInFlightFences[currentFrame]) != VK_SUCCESS) {
    throw std::runtime_error("failed to submit compute command buffer!");
};

// 그래픽 제출
vkWaitForFences(device, 1, &inFlightFences[currentFrame], VK_TRUE, UINT64_MAX);

...

vkResetFences(device, 1, &inFlightFences[currentFrame]);

vkResetCommandBuffer(commandBuffers[currentFrame], /*VkCommandBufferResetFlagBits*/ 0);
recordCommandBuffer(commandBuffers[currentFrame], imageIndex);

VkSemaphore waitSemaphores[] = { computeFinishedSemaphores[currentFrame], imageAvailableSemaphores[currentFrame] };
VkPipelineStageFlags waitStages[] = { VK_PIPELINE_STAGE_VERTEX_INPUT_BIT, VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT };
submitInfo = {};
submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;

submitInfo.waitSemaphoreCount = 2;
submitInfo.pWaitSemaphores = waitSemaphores;
submitInfo.pWaitDstStageMask = waitStages;
submitInfo.commandBufferCount = 1;
submitInfo.pCommandBuffers = &commandBuffers[currentFrame];
submitInfo.signalSemaphoreCount = 1;
submitInfo.pSignalSemaphores = &renderFinishedSemaphores[currentFrame];

if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, inFlightFences[currentFrame]) != VK_SUCCESS) {
    throw std::runtime_error("failed to submit draw command buffer!");
}
```

[세마포어 장](03_Drawing_a_triangle/03_Drawing/02_Rendering_and_presentation.md#page_Semaphores)의 샘플과 비슷한 설정으로, 이 설정은 `vkWaitForFences` 명령을 사용하여 현재 프레임의 컴퓨트 명령 버퍼 실행을 기다린 후에 컴퓨트 제출을 즉시 실행합니다.

그래픽 제출은 컴퓨트 작업이 끝날 때까지 기다려야 하므로 컴퓨트 버퍼가 아직 입자를 업데이트하는 동안 시작하지 않도록 합니다. 따라서 현재 프레임의 `computeFinishedSemaphores`에서 기다리고 그래픽 제출이 `VK_PIPELINE_STAGE_VERTEX_INPUT_BIT` 단계에서 정점을 가져올 때까지 기다립니다.

그러나 프레젠테이션을 위해 기다려야 하므로 프래그먼트 셰이더가 이미지가 표시될 때까지 색상 첨부 파일에 출력하지 않도록 합니다. 따라서 현재 프레임의 `imageAvailableSemaphores`에서 `VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT` 단계에서도 기다립니다.

## 입자 시스템 그리기

앞서 배운 것처럼 Vulkan에서 버퍼는 여러 용도로 사용될 수 있으므로 입자를 포함하는 셰이더 저장 버퍼를 셰이더 저장 버퍼 비트와 정점 버퍼 비트 모두로 생성했습니다. 이는 이전 장에서 "순수" 정점 버퍼를 사용했던 것처럼 셰이더 저장 버퍼를 그리는 데

 사용할 수 있음을 의미합니다.

우리는 입자 구조와 일치하도록 정점 입력 상태를 설정합니다:

```cpp
struct Particle {
    ...

    static std::array<VkVertexInputAttributeDescription, 2> getAttributeDescriptions() {
        std::array<VkVertexInputAttributeDescription, 2> attributeDescriptions{};

        attributeDescriptions[0].binding = 0;
        attributeDescriptions[0].location = 0;
        attributeDescriptions[0].format = VK_FORMAT_R32G32_SFLOAT;
        attributeDescriptions[0].offset = offsetof(Particle, position);

        attributeDescriptions[1].binding = 0;
        attributeDescriptions[1].location = 1;
        attributeDescriptions[1].format = VK_FORMAT_R32G32B32A32_SFLOAT;
        attributeDescriptions[1].offset = offsetof(Particle, color);

        return attributeDescriptions;
    }
};
```

`velocity`는 컴퓨트 셰이더에서만 사용되기 때문에 정점 입력 속성에 추가하지 않습니다.

그런 다음 우리는 모든 정점 버퍼와 마찬가지로 바인딩하고 그립니다:

```cpp
vkCmdBindVertexBuffers(commandBuffer, 0, 1, &shaderStorageBuffer[currentFrame], offsets);

vkCmdDraw(commandBuffer, PARTICLE_COUNT, 1, 0, 0);
```

## 결론

이 장에서 우리는 컴퓨트 셰이더를 사용하여 CPU에서 GPU로 작업을 오프로드하는 방법을 배웠습니다. 컴퓨트 셰이더가 없다면 많은 현대 게임과 애플리케이션에서 많은 효과가 불가능하거나 훨씬 느리게 실행될 것입니다. 그러나 그래픽보다 훨씬 많은 사용 사례가 있으며, 이 장은 가능한 것들의 일부만 보여줍니다. 따라서 이제 컴퓨트 셰이더 사용 방법을 알게 되었으니, 다음과 같은 몇 가지 고급 컴퓨트 주제를 살펴볼 수 있습니다:

- 공유 메모리
- [비동기 컴퓨트](https://github.com/KhronosGroup/Vulkan-Samples/tree/master/samples/performance/async_compute)
- 원자 연산
- [서브그룹](https://www.khronos.org/blog/vulkan-subgroup-tutorial)

[공식 Khronos Vulkan 샘플 저장소](https://github.com/KhronosGroup/Vulkan-Samples/tree/master/samples/api)에서 몇 가지 고급 컴퓨트 샘플을 찾을 수 있습니다.

[C++ 코드](/code/31_compute_shader.cpp) /
[버텍스 셰이더](/code/31_shader_compute.vert) /
[프래그먼트 셰이더](/code/31_shader_compute.frag) /
[컴퓨트 셰이더](/code/31_shader_compute.comp)