# 버텍스 입력 구조 설명

## 소개

이어지는 몇 장에서, 우리는 버텍스 셰이더의 하드코딩된 버텍스 데이터를 메모리에 있는 버텍스 버퍼로 대체할 것입니다. 우선 가장 쉬운 방법인 CPU 가시 버퍼를 생성하고 `memcpy`를 사용하여 버텍스 데이터를 직접 복사하는 방법부터 시작하고, 이후에 고성능 메모리로 버텍스 데이터를 복사하기 위한 스테이징 버퍼 사용 방법을 살펴볼 것입니다.

## 버텍스 셰이더(Vertex Shader)

먼저 버텍스 셰이더 코드에서 버텍스 데이터를 제거하여 셰이더 자체에 버텍스 데이터가 포함되지 않도록 변경합니다. 버텍스 셰이더는 `in` 키워드를 사용하여 버텍스 버퍼로부터 데이터를 받아들입니다.

```glsl
#version 450

layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;

layout(location = 0) out vec3 fragColor;

void main() {
    gl_Position = vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```

`inPosition`과 `inColor`는 버텍스 속성입니다. 이들은 버텍스 버퍼에서 버텍스별로 지정된 속성으로, 우리가 두 배열을 사용해 위치와 색상을 각 버텍스에 수동으로 지정했던 것처럼 작동합니다. 버텍스 셰이더를 재컴파일하는 것을 잊지 마십시오!

`fragColor`와 같이, `layout(location = x)` 주석은 나중에 참조하기 위해 입력에 인덱스를 할당합니다. `dvec3`와 같은 일부 유형은 여러 슬롯을 사용합니다. 즉, 그 다음 인덱스는 최소 2 이상이어야 합니다:

```glsl
layout(location = 0) in dvec3 inPosition;
layout(location = 2) in vec3 inColor;
```

[OpenGL 위키](https://www.khronos.org/opengl/wiki/Layout_Qualifier_(GLSL))에서 레이아웃 한정자에 대한 자세한 정보를 찾을 수 있습니다.

## 버텍스 데이터

우리는 셰이더 코드에서 프로그램 코드의 배열로 버텍스 데이터를 이동합니다. GLM 라이브러리를 포함하여 시작하세요. 이 라이브러리는 벡터와 행렬과 같은 선형 대수 관련 유형을 제공합니다. 이 유형들을 사용하여 위치와 색상 벡터를 지정할 것입니다.

```c++
#include <glm/glm.hpp>
```

버텍스 셰이더에서 사용될 두 요소가 포함된 `Vertex`라는 새로운 구조체를 생성합니다:

```c++
struct Vertex {
    glm::vec2 pos;
    glm::vec3 color;
};
```

GLM은 셰이더 언어에서 사용되는 벡터 유형과 정확히 일치하는 C++ 유형을 편리하게 제공합니다.

```c++
const std::vector<Vertex> vertices = {
    {{0.0f, -0.5f}, {1.0f, 0.0f, 0.0f}},
    {{0.5f, 0.5f}, {0.0f, 1.0f, 0.0f}},
    {{-0.5f, 0.5f}, {0.0f, 0.0f, 1.0f}}
};
```

이제 `Vertex` 구조를 사용하여 버텍스 데이터 배열을 지정합니다. 이전과 동일한 위치와 색상 값을 사용하지만, 이제 이들을 버텍스 요소를 인터리빙하는 하나의 배열로 결합했습니다.

## 바인딩 및 속성 구조 정의

다음 단계는 이 데이터 형식을 GPU 메모리에 업로드한 후 버텍스 셰이더로 전달하는 방법을 Vulkan에 알리는 것입니다. 이 정보를 전달하는 데 필요한 두 가지 유형의 구조체가 있습니다.

첫 번째 구조체는 `VkVertexInputBindingDescription`이며, `Vertex` 구조에 멤버 함수를 추가하여 이 구조를 올바르게 채웁니다.

```c++
struct Vertex {
    glm::vec2 pos;
    glm::vec3 color;

    static VkVertexInputBindingDescription getBindingDescription() {
        VkVertexInputBindingDescription bindingDescription{};

        return bindingDescription;
    }
};
```

버텍스 바인딩은 버텍스를 통해 메모리에서 데이터를 로드하는 비율을 설명합니다. 데이터 항목 간의 바이트 수와 각 버텍스나 각 인스턴스 후에 다음 데이터 항목으로 이동할지를 지정합니다.

```c++
VkVertexInputBindingDescription bindingDescription{};
bindingDescription.binding = 0;
bindingDescription.stride = sizeof(Vertex);
bindingDescription.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;
```

모든 버텍스당 데이터는 하나의 배열에 패키지되어 있으므로 하나의 바인딩만 가질 것입니다. `binding` 매개변수는 바인딩 배열의 인덱스를 지정하고, `stride` 매개변수는 한 항목에서 다음 항목까지의 바이트 수를 지정하며, `inputRate` 매개변수는 다음 값 중 하나를 가질 수 있습니다:

* `VK_VERTEX_INPUT_RATE_VERTEX`: 각 버텍스 후에 다음 데이터 항목으로 이동
* `VK_VERTEX_INPUT_RATE_INSTANCE`: 각 인스턴스 후에 다음 데이터 항목으로 이동

우리는 인스턴스 렌더링을 사용하지 않을 것이므로 버텍스당 데이터를 사용할 것입니다.

## 속성 설명

버텍스 입력을 처리하는 방법을 설명하는 두 번째 구조는 `VkVertexInputAttributeDescription`입니다. 우리는 `Vertex`에 또 다른 도우미 함수를 추가하여 이 구조체들을 채울 것입니다.

```c++
#include <array>

...

static std::array<VkVertexInputAttributeDescription, 2> getAttributeDescriptions() {
    std::array<VkVertexInputAttributeDescription, 2> attributeDescriptions{};

    return attributeDescriptions;
}
```

함수 프로토타입이 나타내듯이 이 구조체는 두 개가 될 것입니다. 속성 설명 구조체는 바인딩 설명에서 기원하는 버텍스 데이터 덩어리에서 버텍스 속성을 추출하는 방법을 설명합니다. 우리는 위치와 색상의 두 속성을 가지고 있으므로 두 개의 속성 설명 구조체가 필요합니다.

```c++
attributeDescriptions[0].binding = 0;
attributeDescriptions[0].location = 0;
attributeDescriptions[0].format = VK_FORMAT_R32G32_SFLOAT;
attributeDescriptions[0].offset = offsetof(Vertex, pos);
```

`binding` 매개변수는 버텍스당 데이터가 어디에서 오는지 Vulkan에 알려줍니다. `location` 매개변수는 버텍스 셰이더의 입력의 `location` 지시문을 참조합니다. 버텍스 셰이더의 위치 `0`에 있는 입력은 위치이며, 32비트 float

의 두 구성 요소를 가지고 있습니다.

`format` 매개변수는 속성 데이터의 유형을 설명합니다. 약간 혼란스럽게도, 형식은 색상 형식과 동일한 열거를 사용하여 지정됩니다. 일반적으로 함께 사용되는 셰이더 유형과 형식은 다음과 같습니다:

* `float`: `VK_FORMAT_R32_SFLOAT`
* `vec2`: `VK_FORMAT_R32G32_SFLOAT`
* `vec3`: `VK_FORMAT_R32G32B32_SFLOAT`
* `vec4`: `VK_FORMAT_R32G32B32A32_SFLOAT`

보시다시피, 색상 채널의 수가 셰이더 데이터 유형의 구성 요소 수와 일치하는 형식을 사용해야 합니다. 채널 수가 셰이더의 구성 요소 수보다 많을 경우 허용되지만, 그러한 경우 추가 채널은 조용히 버려집니다. 채널 수가 구성 요소 수보다 적은 경우, BGA 구성 요소는 기본값 `(0, 0, 1)`을 사용합니다. 색상 유형(`SFLOAT`, `UINT`, `SINT`) 및 비트 폭도 셰이더 입력의 유형과 일치해야 합니다. 다음 예를 참조하십시오:

* `ivec2`: `VK_FORMAT_R32G32_SINT`, 32비트 부호 있는 정수의 2-구성 요소 벡터
* `uvec4`: `VK_FORMAT_R32G32B32A32_UINT`, 32비트 부호 없는 정수의 4-구성 요소 벡터
* `double`: `VK_FORMAT_R64_SFLOAT`, 더블 정밀도(64비트) float

`format` 매개변수는 속성 데이터의 바이트 크기를 암시적으로 정의하고 `offset` 매개변수는 버텍스당 데이터의 시작부터 읽을 바이트 수를 지정합니다. 바인딩은 한 번에 하나의 `Vertex`를 로드하고 위치 속성(`pos`)은 이 구조체의 시작부터 `0` 바이트 오프셋에 있습니다. 이는 `offsetof` 매크로를 사용하여 자동으로 계산됩니다.

```c++
attributeDescriptions[1].binding = 0;
attributeDescriptions[1].location = 1;
attributeDescriptions[1].format = VK_FORMAT_R32G32B32_SFLOAT;
attributeDescriptions[1].offset = offsetof(Vertex, color);
```

색상 속성도 거의 같은 방식으로 설명됩니다.

## 파이프라인 버텍스 입력

이제 그래픽 파이프라인을 설정하여 이 형식의 버텍스 데이터를 수락하고 버텍스 셰이더로 전달할 수 있도록 `createGraphicsPipeline`에서 구조체를 참조해야 합니다. `vertexInputInfo` 구조체를 찾아 두 설명을 참조하도록 수정하세요:

```c++
auto bindingDescription = Vertex::getBindingDescription();
auto attributeDescriptions = Vertex::getAttributeDescriptions();

vertexInputInfo.vertexBindingDescriptionCount = 1;
vertexInputInfo.vertexAttributeDescriptionCount = static_cast<uint32_t>(attributeDescriptions.size());
vertexInputInfo.pVertexBindingDescriptions = &bindingDescription;
vertexInputInfo.pVertexAttributeDescriptions = attributeDescriptions.data();
```

파이프라인은 이제 `vertices` 컨테이너의 형식의 버텍스 데이터를 수락하고 버텍스 셰이더로 전달할 준비가 되었습니다. 이제 프로그램을 실행하면 유효성 검사 계층이 활성화되어 있고 바인딩에 버텍스 버퍼가 바인딩되어 있지 않다고 불평하는 것을 볼 수 있습니다. 다음 단계는 버텍스 버퍼를 생성하고 버텍스 데이터를 그곳으로 이동하여 GPU가 접근할 수 있도록 하는 것입니다.

[C++ 코드](/code/18_vertex_input.cpp) /
[버텍스 셰이더](/code/18_shader_vertexbuffer.vert) /
[프래그먼트 셰이더](/code/18_shader_vertexbuffer.frag)