# 셰이더 모듈

이전 API들과 달리, Vulkan의 셰이더 코드는 [GLSL](https://en.wikipedia.org/wiki/OpenGL_Shading_Language)이나 [HLSL](https://en.wikipedia.org/wiki/High-Level_Shading_Language)과 같은 사람이 읽을 수 있는 문법이 아닌 바이트코드 형식으로 지정해야 합니다. 이 바이트코드 형식을 [SPIR-V](https://www.khronos.org/spir)라고 하며, Vulkan과 OpenCL(둘 다 Khronos API) 모두에서 사용할 수 있도록 설계되었습니다. 이는 그래픽스와 컴퓨트 셰이더를 작성하는 데 사용할 수 있는 형식이지만, 이 튜토리얼에서는 Vulkan의 그래픽스 파이프라인에서 사용되는 셰이더에 초점을 맞출 것입니다.

바이트코드 형식을 사용하는 장점은 GPU 벤더들이 셰이더 코드를 네이티브 코드로 변환하기 위해 작성한 컴파일러가 훨씬 덜 복잡하다는 것입니다. 과거에는 GLSL과 같은 사람이 읽을 수 있는 문법을 사용할 때 일부 GPU 벤더들이 표준을 해석하는 데 있어 다소 유연했습니다. 이러한 벤더의 GPU로 복잡한 셰이더를 작성하게 되면, 다른 벤더의 드라이버가 문법 오류로 인해 코드를 거부하거나, 더 나쁜 경우에는 컴파일러 버그로 인해 셰이더가 다르게 실행될 위험이 있었습니다. SPIR-V와 같은 간단한 바이트코드 형식을 사용하면 이러한 문제를 피할 수 있을 것입니다.

하지만 이것이 우리가 이 바이트코드를 직접 작성해야 한다는 의미는 아닙니다. Khronos는 GLSL을 SPIR-V로 컴파일하는 자체 벤더 독립적인 컴파일러를 출시했습니다. 이 컴파일러는 셰이더 코드가 완전히 표준을 준수하는지 확인하고 프로그램과 함께 제공할 수 있는 하나의 SPIR-V 바이너리를 생성하도록 설계되었습니다. 또한 이 컴파일러를 라이브러리로 포함하여 런타임에 SPIR-V를 생성할 수도 있지만, 이 튜토리얼에서는 그렇게 하지 않을 것입니다. `glslangValidator.exe`를 통해 이 컴파일러를 직접 사용할 수 있지만, 대신 Google의 `glslc.exe`를 사용할 것입니다. `glslc`의 장점은 GCC와 Clang과 같은 잘 알려진 컴파일러와 동일한 매개변수 형식을 사용하고 *include*와 같은 추가 기능을 포함한다는 것입니다. 두 컴파일러 모두 Vulkan SDK에 이미 포함되어 있으므로 추가로 다운로드할 필요가 없습니다.

GLSL은 C 스타일 문법을 가진 셰이딩 언어입니다. 이 언어로 작성된 프로그램은 모든 객체에 대해 호출되는 `main` 함수를 가집니다. 입력에 매개변수를 사용하고 출력에 반환 값을 사용하는 대신, GLSL은 입력과 출력을 처리하기 위해 전역 변수를 사용합니다. 이 언어는 내장 벡터와 행렬 기본 타입과 같이 그래픽스 프로그래밍을 돕는 많은 기능을 포함합니다. 외적, 행렬-벡터 곱, 벡터에 대한 반사와 같은 연산을 위한 함수들이 포함되어 있습니다. 벡터 타입은 요소의 수를 나타내는 숫자가 붙은 `vec`로 불립니다. 예를 들어, 3D 위치는 `vec3`에 저장됩니다. `.x`와 같은 멤버를 통해 단일 컴포넌트에 접근할 수 있으며, 동시에 여러 컴포넌트로부터 새로운 벡터를 생성하는 것도 가능합니다. 예를 들어, `vec3(1.0, 2.0, 3.0).xy` 표현식은 `vec2`를 결과로 합니다. 벡터의 생성자는 벡터 객체와 스칼라 값의 조합도 받을 수 있습니다. 예를 들어, `vec3`는 `vec3(vec2(1.0, 2.0), 3.0)`으로 생성될 수 있습니다.

이전 장에서 언급했듯이, 화면에 삼각형을 그리기 위해서는 버텍스 셰이더(vertex shader)와 프래그먼트 셰이더(fragment shader)를 작성해야 합니다. 다음 두 섹션에서는 각각의 GLSL 코드를 다룰 것이며, 그 후에 두 개의 SPIR-V 바이너리를 생성하고 프로그램에 로드하는 방법을 보여드리겠습니다.

## 버텍스 셰이더 (vertex shader)

버텍스 셰이더는 들어오는 각 정점(Vertex)을 처리합니다. 모델 공간 위치, 색상, 법선, 텍스처 좌표와 같은 속성을 입력으로 받습니다. 출력은 클립 좌표계의 최종 위치와 색상, 텍스처 좌표와 같이 프래그먼트 셰이더로 전달되어야 하는 속성들입니다. 이러한 값들은 래스터라이저에 의해 프래그먼트들 사이에서 보간되어 부드러운 그라데이션을 만들어냅니다.

*클립 좌표*는 버텍스 셰이더에서 나온 4차원 벡터로, 전체 벡터를 마지막 컴포넌트로 나누어 *정규화된 장치 좌표*로 변환됩니다. 이 정규화된 장치 좌표는 프레임버퍼를 [-1, 1] × [-1, 1] 좌표계에 매핑하는 [동차 좌표](https://en.wikipedia.org/wiki/Homogeneous_coordinates)로, 다음과 같습니다:

![](/images/normalized_device_coordinates.svg)

컴퓨터 그래픽스를 다뤄본 적이 있다면 이미 이것에 익숙할 것입니다. OpenGL을 사용해본 적이 있다면, Y 좌표의 부호가 이제 반전되었다는 것을 알 수 있습니다. Z 좌표는 이제 Direct3D에서처럼 0에서 1 사이의 범위를 사용합니다.

우리의 첫 번째 삼각형에서는 어떤 변환도 적용하지 않을 것이며, 세 정점의 위치를 정규화된 장치 좌표로 직접 지정하여 다음과 같은 모양을 만들 것입니다:

![](/images/triangle_coordinates.svg)

마지막 컴포넌트를 `1`로 설정하여 버텍스 셰이더에서 클립 좌표로 직접 출력함으로써 정규화된 장치 좌표를 직접 출력할 수 있습니다. 이렇게 하면 클립 좌표를 정규화된 장치 좌표로 변환하는 나눗셈이 아무것도 변경하지 않을 것입니다.

일반적으로 이러한 좌표들은 버텍스 버퍼(vertex buffer)에 저장되지만, Vulkan에서 버텍스 버퍼를 생성하고 데이터를 채우는 것은 간단하지 않습니다. 따라서 화면에 삼각형이 나타나는 것을 보는 즐거움을 느낀 후로 이를 미루기로 했습니다. 그동안 우리는 조금 비정통적인 방법을 사용할 것입니다: 좌표를 버텍스 셰이더 안에 직접 포함시키는 것입니다. 코드는 다음과 같습니다:

```glsl
#version 450

vec2 positions[3] = vec2[](
    vec2(0.0, -0.5),
    vec2(0.5, 0.5),
    vec2(-0.5, 0.5)
);

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
}
```

`main` 함수는 모든 정점에 대해 호출됩니다. 내장 변수 `gl_VertexIndex`는 현재 정점의 인덱스를 포함합니다. 이는 보통 버텍스 버퍼의 인덱스이지만, 우리의 경우에는 하드코딩된 버텍스 데이터 배열의 인덱스가 될 것입니다. 각 정점의 위치는 셰이더의 상수 배열에서 접근되어 더미 `z`와 `w` 컴포넌트와 결합되어 클립 좌표의 위치를 생성합니다. 내장 변수 `gl_Position`이 출력으로 기능합니다.

## 프래그먼트 셰이더 (Fragment Shader)

버텍스 셰이더의 위치들로 형성된 삼각형은 화면상의 영역을 프래그먼트로 채웁니다. 프래그먼트 셰이더는 이러한 프래그먼트들에 대해 호출되어 프레임버퍼(또는 프레임버퍼들)에 대한 색상과 깊이를 생성합니다. 전체 삼각형을 빨간색으로 출력하는 간단한 프래그먼트 셰이더는 다음과 같습니다:

```glsl
#version 450

layout(location = 0) out vec4 outColor;

void main() {
    outColor = vec4(1.0, 0.0, 0.0, 1.0);
}
```

버텍스 셰이더의 `main` 함수가 모든 정점에 대해 호출되는 것처럼, `main` 함수는 모든 프래그먼트에 대해 호출됩니다. GLSL에서 색상은 [0, 1] 범위 내의 R, G, B와 알파 채널을 가진 4-컴포넌트 벡터입니다. 버텍스 셰이더의 `gl_Position`과 달리, 현재 프래그먼트의 색상을 출력하기 위한 내장 변수는 없습니다. 각 프레임버퍼에 대해 자신만의 출력 변수를 지정해야 하며, 여기서 `layout(location = 0)` 수식자는 프레임버퍼의 인덱스를 지정합니다. 빨간색은 인덱스 `0`의 첫 번째(그리고 유일한) 프레임버퍼에 연결된 이 `outColor` 변수에 기록됩니다.

## 정점별 색상

전체 삼각형을 빨간색으로 만드는 것은 그다지 흥미롭지 않습니다. 다음과 같은 것이 더 보기 좋지 않을까요?

![](/images/triangle_coordinates_colors.png)

이를 구현하기 위해 두 셰이더 모두에 몇 가지 변경을 해야 합니다. 먼저, 세 정점 각각에 대해 서로 다른 색상을 지정해야 합니다. 버텍스 셰이더는 이제 위치에 대한 배열처럼 색상에 대한 배열도 포함해야 합니다:

```glsl
vec3 colors[3] = vec3[](
    vec3(1.0, 0.0, 0.0),
    vec3(0.0, 1.0, 0.0),
    vec3(0.0, 0.0, 1.0)
);
```

이제 이러한 정점별 색상을 프래그먼트 셰이더로 전달하여 보간된 값을 프레임버퍼에 출력할 수 있도록 해야 합니다. 버텍스 셰이더에 색상에 대한 출력을 추가하고 `main` 함수에서 이를 기록합니다:

```glsl
layout(location = 0) out vec3 fragColor;

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
    fragColor = colors[gl_VertexIndex];
}
```

다음으로, 프래그먼트 셰이더에 매칭되는 입력을 추가해야 합니다:

```glsl
layout(location = 0) in vec3 fragColor;

void main() {
    outColor = vec4(fragColor, 1.0);
}
```

입력 변수가 반드시 같은 이름을 사용할 필요는 없습니다. `location` 지시자로 지정된 인덱스를 사용하여 서로 연결됩니다. `main` 함수는 알파 값과 함께 색상을 출력하도록 수정되었습니다. 위 이미지에서 보이듯이, `fragColor`의 값은 세 정점 사이의 프래그먼트들에 대해 자동으로 보간되어 부드러운 그라데이션을 만들어냅니다.

## 셰이더 컴파일하기

프로젝트의 루트 디렉토리에 `shaders`라는 디렉토리를 만들고, 그 디렉토리에 버텍스 셰이더를 `shader.vert` 파일에, 프래그먼트 셰이더를 `shader.frag` 파일에 저장하세요. GLSL 셰이더는 공식 확장자가 없지만, 이 두 가지가 일반적으로 구분을 위해 사용됩니다.

`shader.vert`의 내용은 다음과 같아야 합니다:

```glsl
#version 450

layout(location = 0) out vec3 fragColor;

vec2 positions[3] = vec2[](
    vec2(0.0, -0.5),
    vec2(0.5, 0.5),
    vec2(-0.5, 0.5)
);

vec3 colors[3] = vec3[](
    vec3(1.0, 0.0, 0.0),
    vec3(0.0, 1.0, 0.0),
    vec3(0.0, 0.0, 1.0)
);

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
    fragColor = colors[gl_VertexIndex];
}
```

그리고 `shader.frag`의 내용은 다음과 같아야 합니다:

```glsl
#version 450

layout(location = 0) in vec3 fragColor;

layout(location = 0) out vec4 outColor;

void main() {
    outColor = vec4(fragColor, 1.0);
}
```

이제 `glslc` 프로그램을 사용하여 이들을 SPIR-V 바이트코드로 컴파일하겠습니다.

**Windows**

다음 내용으로 `compile.bat` 파일을 만드세요:

```bash
C:/VulkanSDK/x.x.x.x/Bin/glslc.exe shader.vert -o vert.spv
C:/VulkanSDK/x.x.x.x/Bin/glslc.exe shader.frag -o frag.spv
pause
```

Vulkan SDK를 설치한 경로로 `glslc.exe`의 경로를 바꾸세요. 파일을 더블 클릭하여 실행하세요.

**Linux**

다음 내용으로 `compile.sh` 파일을 만드세요:

```bash
/home/user/VulkanSDK/x.x.x.x/x86_64/bin/glslc shader.vert -o vert.spv
/home/user/VulkanSDK/x.x.x.x/x86_64/bin/glslc shader.frag -o frag.spv
```

Vulkan SDK를 설치한 경로로 `glslc`의 경로를 바꾸세요. `chmod +x compile.sh`로 스크립트를 실행 가능하게 만들고 실행하세요.

**플랫폼별 지침 끝**

이 두 명령어는 컴파일러에게 GLSL 소스 파일을 읽어서 `-o` (출력) 플래그를 사용해 SPIR-V 바이트코드 파일을 출력하도록 지시합니다.

셰이더에 구문 오류가 있다면 컴파일러는 예상대로 줄 번호와 문제점을 알려줄 것입니다. 예를 들어 세미콜론을 생략하고 컴파일 스크립트를 다시 실행해보세요. 또한 컴파일러를 인자 없이 실행해보면 어떤 종류의 플래그들을 지원하는지 볼 수 있습니다. 예를 들어, 바이트코드를 사람이 읽을 수 있는 형식으로 출력할 수도 있어서 셰이더가 정확히 무엇을 하는지, 그리고 이 단계에서 어떤 최적화가 적용되었는지 확인할 수 있습니다.

명령줄에서 셰이더를 컴파일하는 것이 가장 간단한 방법 중 하나이며, 이 튜토리얼에서도 이 방법을 사용할 것입니다. 하지만 자신의 코드에서 직접 셰이더를 컴파일하는 것도 가능합니다. Vulkan SDK는 프로그램 내에서 GLSL 코드를 SPIR-V로 컴파일할 수 있는 라이브러리인 [libshaderc](https://github.com/google/shaderc)를 포함하고 있습니다.

## 셰이더 로딩하기

이제 SPIR-V 셰이더를 생성할 수 있게 되었으니, 그래픽스 파이프라인에 연결하기 위해 프로그램에 로드할 차례입니다. 먼저 파일에서 바이너리 데이터를 로드하는 간단한 헬퍼 함수를 작성해보겠습니다.

```c++
#include <fstream>

...

static std::vector<char> readFile(const std::string& filename) {
    std::ifstream file(filename, std::ios::ate | std::ios::binary);

    if (!file.is_open()) {
        throw std::runtime_error("failed to open file!");
    }
}
```

`readFile` 함수는 지정된 파일에서 모든 바이트를 읽어서 `std::vector`가 관리하는 바이트 배열로 반환합니다. 우리는 두 가지 플래그를 사용하여 파일을 엽니다:

* `ate`: 파일의 끝에서부터 읽기 시작
* `binary`: 파일을 바이너리 파일로 읽기 (텍스트 변환 방지)

파일의 끝에서 읽기를 시작하는 것의 장점은 읽기 위치를 통해 파일의 크기를 확인하고 버퍼를 할당할 수 있다는 것입니다:

```c++
size_t fileSize = (size_t) file.tellg();
std::vector<char> buffer(fileSize);
```

그런 다음, 파일의 시작 위치로 되돌아가서 모든 바이트를 한 번에 읽을 수 있습니다:

```c++
file.seekg(0);
file.read(buffer.data(), fileSize);
```

마지막으로 파일을 닫고 바이트들을 반환합니다:

```c++
file.close();

return buffer;
```

이제 `createGraphicsPipeline`에서 이 함수를 호출하여 두 셰이더의 바이트코드를 로드하겠습니다:

```c++
void createGraphicsPipeline() {
    auto vertShaderCode = readFile("shaders/vert.spv");
    auto fragShaderCode = readFile("shaders/frag.spv");
}
```

버퍼의 크기를 출력하고 실제 파일 크기(바이트)와 일치하는지 확인하여 셰이더가 올바르게 로드되었는지 확인하세요. 바이너리 코드이고 나중에 크기를 명시적으로 지정할 것이기 때문에 코드가 null로 끝날 필요는 없습니다.

## 셰이더 모듈 생성하기

파이프라인에 코드를 전달하기 전에, 이를 `VkShaderModule` 객체로 래핑해야 합니다. 이를 위한 헬퍼 함수 `createShaderModule`을 만들어보겠습니다.

```c++
VkShaderModule createShaderModule(const std::vector<char>& code) {

}
```

이 함수는 바이트코드가 있는 버퍼를 매개변수로 받아 `VkShaderModule`을 생성합니다.

셰이더 모듈을 생성하는 것은 간단합니다. 바이트코드가 있는 버퍼에 대한 포인터와 그 길이만 지정하면 됩니다. 이 정보는 `VkShaderModuleCreateInfo` 구조체에 지정됩니다. 한 가지 주의할 점은 바이트코드의 크기는 바이트 단위로 지정되지만, 바이트코드 포인터는 `char` 포인터가 아닌 `uint32_t` 포인터여야 한다는 것입니다. 따라서 아래와 같이 `reinterpret_cast`를 사용하여 포인터를 캐스팅해야 합니다. 이런 캐스팅을 수행할 때는 데이터가 `uint32_t`의 정렬 요구사항을 만족하는지 확인해야 합니다. 다행히도 데이터는 `std::vector`에 저장되어 있고, 기본 할당자가 이미 최악의 경우의 정렬 요구사항을 만족하도록 보장합니다.

```c++
VkShaderModuleCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_SHADER_MODULE_CREATE_INFO;
createInfo.codeSize = code.size();
createInfo.pCode = reinterpret_cast<const uint32_t*>(code.data());
```

그런 다음 `vkCreateShaderModule`을 호출하여 `VkShaderModule`을 생성할 수 있습니다:

```c++
VkShaderModule shaderModule;
if (vkCreateShaderModule(device, &createInfo, nullptr, &shaderModule) != VK_SUCCESS) {
    throw std::runtime_error("failed to create shader module!");
}
```

매개변수들은 이전의 객체 생성 함수들과 동일합니다: 논리적 디바이스, 생성 정보 구조체에 대한 포인터, 선택적 커스텀 할당자에 대한 포인터, 그리고 핸들 출력 변수입니다. 셰이더 모듈을 생성한 후에는 바로 코드가 담긴 버퍼를 해제할 수 있습니다. 생성된 셰이더 모듈을 반환하는 것을 잊지 마세요:

```c++
return shaderModule;
```

셰이더 모듈은 우리가 이전에 파일에서 로드한 셰이더 바이트코드와 그 안에 정의된 함수들을 감싸는 얇은 래퍼일 뿐입니다. SPIR-V 바이트코드를 GPU에서 실행할 수 있는 기계어로 컴파일하고 링크하는 것은 그래픽스 파이프라인이 생성될 때까지 일어나지 않습니다. 이는 파이프라인 생성이 완료되는 즉시 셰이더 모듈을 파괴할 수 있다는 것을 의미하며, 이것이 바로 우리가 이들을 클래스 멤버가 아닌 `createGraphicsPipeline` 함수의 지역 변수로 만드는 이유입니다:

```c++
void createGraphicsPipeline() {
    auto vertShaderCode = readFile("shaders/vert.spv");
    auto fragShaderCode = readFile("shaders/frag.spv");

    VkShaderModule vertShaderModule = createShaderModule(vertShaderCode);
    VkShaderModule fragShaderModule = createShaderModule(fragShaderCode);
```

그런 다음 함수의 끝에서 `vkDestroyShaderModule`을 두 번 호출하여 정리해야 합니다. 이 장의 나머지 코드는 모두 이 라인들 앞에 삽입될 것입니다.

```c++
    ...
    vkDestroyShaderModule(device, fragShaderModule, nullptr);
    vkDestroyShaderModule(device, vertShaderModule, nullptr);
}
```

## 셰이더 스테이지 생성

셰이더를 실제로 사용하기 위해서는 파이프라인 생성 과정의 일부로 `VkPipelineShaderStageCreateInfo` 구조체를 통해 특정 파이프라인 스테이지에 할당해야 합니다.

`createGraphicsPipeline` 함수에서 버텍스 셰이더를 위한 구조체를 먼저 채워보겠습니다.

```c++
VkPipelineShaderStageCreateInfo vertShaderStageInfo{};
vertShaderStageInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
vertShaderStageInfo.stage = VK_SHADER_STAGE_VERTEX_BIT;
```

필수적인 `sType` 멤버 외에도, 첫 번째 단계는 Vulkan에게 파이프라인의 어느 스테이지에서 셰이더가 사용될 것인지 알려주는 것입니다. 이전 장에서 설명한 각각의 프로그래밍 가능한 스테이지에 대한 열거형 값이 있습니다.

```c++
vertShaderStageInfo.module = vertShaderModule;
vertShaderStageInfo.pName = "main";
```

다음 두 멤버는 코드가 포함된 셰이더 모듈과, *엔트리포인트*라고 알려진 호출할 함수를 지정합니다. 이는 여러 개의 프래그먼트 셰이더를 하나의 셰이더 모듈로 결합하고 서로 다른 엔트리 포인트를 사용하여 각각의 동작을 구분할 수 있다는 것을 의미합니다. 이 경우에는 표준인 `main`을 사용하겠습니다.

여기서 사용하지는 않지만 논의할 가치가 있는 또 하나의 (선택적) 멤버가 있는데, 바로 `pSpecializationInfo`입니다. 이를 통해 셰이더 상수에 대한 값을 지정할 수 있습니다. 하나의 셰이더 모듈을 사용하면서 파이프라인 생성 시점에 모듈에서 사용되는 상수들에 대해 서로 다른 값을 지정함으로써 동작을 구성할 수 있습니다. 이는 렌더링 시점에 변수를 사용하여 셰이더를 구성하는 것보다 더 효율적입니다. 컴파일러가 이러한 값들에 의존하는 `if` 문을 제거하는 등의 최적화를 수행할 수 있기 때문입니다. 만약 그런 상수들이 없다면, 이 멤버를 `nullptr`로 설정할 수 있으며, 우리의 구조체 초기화는 이를 자동으로 수행합니다.

프래그먼트 셰이더에 맞게 구조체를 수정하는 것은 간단합니다:

```c++
VkPipelineShaderStageCreateInfo fragShaderStageInfo{};
fragShaderStageInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
fragShaderStageInfo.stage = VK_SHADER_STAGE_FRAGMENT_BIT;
fragShaderStageInfo.module = fragShaderModule;
fragShaderStageInfo.pName = "main";
```

마지막으로 이 두 구조체를 포함하는 배열을 정의하여, 나중에 실제 파이프라인 생성 단계에서 이들을 참조할 수 있도록 합니다.

```c++
VkPipelineShaderStageCreateInfo shaderStages[] = {vertShaderStageInfo, fragShaderStageInfo};
```

파이프라인의 프로그래밍 가능한 스테이지를 설명하는 것은 이게 전부입니다. 다음 장에서는 고정 기능 스테이지들을 살펴보겠습니다.

[C++ 코드](/code/09_shader_modules.cpp) /
[버텍스 셰이더](/code/09_shader_base.vert) /
[프래그먼트 셰이더](/code/09_shader_base.frag)