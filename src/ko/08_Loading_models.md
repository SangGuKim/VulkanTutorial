다음은 모델 로딩에 대한 번역본입니다. 원본 구조를 그대로 유지하려고 노력했습니다.

---

# 모델 로딩

## 소개

이제 프로그램이 텍스처가 적용된 3D 메시를 렌더링할 준비가 되었지만, 현재 `vertices`와 `indices` 배열에 있는 기하학적 구조는 아직 흥미롭지 않습니다. 이 장에서는 실제 모델 파일에서 정점과 인덱스를 로드하여 그래픽 카드에 실제 작업을 하도록 프로그램을 확장할 것입니다.

많은 그래픽 API 튜토리얼은 독자가 이러한 장에서 자체 OBJ 로더를 작성하도록 합니다. 이 방법의 문제는 어느 정도 흥미로운 3D 애플리케이션이 곧 이 파일 형식에서 지원하지 않는 기능, 예를 들어 골격 애니메이션과 같은 것들을 필요로 한다는 것입니다. 이 장에서는 OBJ 모델에서 메시 데이터를 로드할 것이지만, 파일에서 로딩하는 세부사항보다는 프로그램 자체와 메시 데이터를 통합하는 데 더 초점을 맞출 것입니다.

## 라이브러리

OBJ 파일에서 정점과 면을 로드하기 위해 [tinyobjloader](https://github.com/syoyo/tinyobjloader) 라이브러리를 사용할 것입니다. 이 라이브러리는 stb_image처럼 단일 파일 라이브러리이기 때문에 통합하기 쉽고 빠릅니다. 위에 링크된 저장소로 가서 `tiny_obj_loader.h` 파일을 라이브러리 디렉토리의 폴더에 다운로드하세요.

**Visual Studio**

`tiny_obj_loader.h`가 있는 디렉토리를 `Additional Include Directories` 경로에 추가하세요.

![](/images/include_dirs_tinyobjloader.png)

**Makefile**

GCC의 include 디렉토리에 `tiny_obj_loader.h`가 있는 디렉토리를 추가하세요:

```text
VULKAN_SDK_PATH = /home/user/VulkanSDK/x.x.x.x/x86_64
STB_INCLUDE_PATH = /home/user/libraries/stb
TINYOBJ_INCLUDE_PATH = /home/user/libraries/tinyobjloader

...

CFLAGS = -std=c++17 -I$(VULKAN_SDK_PATH)/include -I$(STB_INCLUDE_PATH) -I$(TINYOBJ_INCLUDE_PATH)
```

## 샘플 메시

이 장에서는 아직 조명을 활성화하지 않을 것이므로, 텍스처에 조명이 베이크된 샘플 모델을 사용하는 것이 도움이 됩니다. 이러한 모델을 찾는 쉬운 방법은 [Sketchfab](https://sketchfab.com/)에서 3D 스캔을 검색하는 것입니다. 그 사이트의 많은 모델들이 관대한 라이선스로 OBJ 형식으로 제공됩니다.

이 튜토리얼을 위해 저는 [Viking room](https://sketchfab.com/3d-models/viking-room-a49f1b8e4f5c4ecf9e1fe7d81915ad38) 모델을 선택했습니다. 이 모델은 [nigelgoh](https://sketchfab.com/nigelgoh)에 의해 만들어졌으며 ([CC BY 4.0](https://web.archive.org/web/20200428202538/https://sketchfab.com/3d-models/viking-room-a49f1b8e4f5c4ecf9e1fe7d81915ad38)) 현재의 기하학적 구조를 대체할 수 있도록 모델의 크기와 방향을 조정했습니다:

* [viking_room.obj](/resources/viking_room.obj)
* [viking_room.png](/resources/viking_room.png)

원하는 모델을 사용할 수 있지만, 한 가지 재질만으로 구성되어 있고 크기가 약 1.5 x 1.5 x 1.5 단위인지 확인하세요. 만약 그보다 크다면, 뷰 행렬을 변경해야 합니다. 모델 파일을 `shaders`와 `textures` 옆의 새로운 `models` 디렉토리에 넣고 텍스처 이미지를 `textures` 디렉토리에 넣으세요.

프로그램에 모델 및 텍스처 경로를 정의하는 두 개의 새로운 구성 변수를 넣으세요:

```c++
const uint32_t WIDTH = 800;
const uint32_t HEIGHT = 600;

const std::string MODEL_PATH = "models/viking_room.obj";
const std::string TEXTURE_PATH = "textures/viking_room.png";
```

그리고 이 경로 변수를 사용하도록 `createTextureImage`를 업데이트하세요:

```c++
stbi_uc* pixels = stbi_load(TEXTURE_PATH.c_str(), &texWidth, &texHeight, &texChannels, STBI_rgb_alpha);
```

## 정점 및 인덱스 로딩

이제 모델 파일에서 정점과 인덱스를 로드할 것이므로, 전역 `vertices` 및 `indices` 배열을 이제 제거하세요. 클래스 멤버로서 비상수 컨테이너로 대체하세요:

```c++
std::vector<Vertex> vertices;
std::vector<uint32_t> indices;
VkBuffer vertexBuffer;
VkDeviceMemory vertexBufferMemory;
```

인덱스의 유형을 `uint16_t`에서 `uint32_t`로 변경해야 합니다. 왜냐하면 65535개 이상의 정점이 있을 것이기 때문입니다. `vkCmdBindIndexBuffer` 매개변수도 변경하세요:

```c++
vkCmdBindIndexBuffer(commandBuffer, indexBuffer, 0, VK_INDEX_TYPE_UINT32);
```

tinyobjloader 라이브러리는 STB 라이브러리와 같은 방식으로 포함됩니다. `tiny_obj_loader.h` 파일을 포함하고 링커 오류를 방지하기 위해 한 소스 파일에서 `TINYOBJLOADER_IMPLEMENTATION`을 정의하세요:

```c++
#define TINYOBJLOADER_IMPLEMENTATION
#include <tiny_obj_loader.h>
```

이제 메시의 정점 데이터로 `vertices` 및 `indices` 컨테이너를 채우기 위해 이 라이브러리를 사용하는 `loadModel` 함수를 작성할 것입니다. 이 함수는 버텍스 및 인덱스 버퍼를 생성하기 전 어딘가에서 호출되어야 합니다:

```c++
void initVulkan() {
    ...
    loadModel();
    createVertexBuffer();
    createIndexBuffer();
    ...
}

...

void loadModel() {

}
```

모델은 `tinyobj::LoadObj` 함수를 호출하여 라이브러리의 데이터 구조로 로드됩니다:

```c++
void loadModel() {
    tinyobj::attrib_t attrib;
    std::vector<tinyobj::shape_t> shapes;
    std::vector<tinyobj::material_t> materials;
    std::string warn, err;

    if (!tinyobj::LoadObj(&attrib, &shapes, &materials, &warn, &err, MODEL_PATH.c_str())) {
        throw std::runtime_error(warn + err);
    }
}
```

OBJ 파일은 위치, 법선, 텍스처 좌표 및 면을 포함합니다. 면은 위치, 법선 및/또는 텍스처 좌표를 인덱스로 참조하는 임의의 양의 정점으로 구성됩니다. 이를 통해 전체 정점뿐만 아니라 개별 속성도 재사용할 수 있습니다.

`attrib` 컨테이너는 `attrib.vertices`, `attrib.normals`, `attrib.texcoords` 벡터에 모든 위치, 법선 및 텍스처 좌표를 보유합니다. `shapes` 컨테이너는 모든 개별 객체와 그 면을 포함합니다. 각 면은 정점 배열로 구성되며, 각 정점에는 위치, 법선 및 텍스처 좌표 속성의 인덱스가 포함됩니다. OBJ 모델은 면마다 재질과 텍스처를 정의할 수도 있지만, 우리는 이를 무시할 것입니다.

`err` 문자열에는 파일을 로딩하는 동안 발생한 오류가 포함되어 있고, `warn` 문자열에는 재질 정의가 누락된 것과 같은 경고가 포함되어 있습니다. 로딩이 실패했다는 것은 `LoadObj` 함수가 `false`를 반환할 때만 해당됩니다. 앞서 언급했듯이, OBJ 파일의 면은 실제로 임의의 수의 정점을 포함할 수 있지만, 우리의 애플리케이션은 삼각형만 렌더링할 수 있습니다. 다행히 `LoadObj`는 이러한 면을 자동으로 삼각형화하는 선택적 매개변수를 제공하며, 기본적으로 활성화되어 있습니다.

파일의 모든 면을 단일 모델로 결합할 것이므로, 모든 형상에 대해 반복하면 됩니다:

```c++
for (const auto& shape : shapes) {

}
```

삼각형화 기능은 이미 면 당 세 개의 정점을 보장했으므로, 이제 정점을 직접 반복하고 우리의 `vertices` 벡터로 직접 넣을 수 있습니다:

```c++
for (const auto& shape : shapes) {
    for (const auto& index : shape.mesh.indices) {
        Vertex vertex{};

        vertices.push_back(vertex);
        indices.push_back(indices.size());
    }
}
```

간단함을 위해 지금은 모든 정점이 고유하다고 가정하므로, 간단한 자동 증분 인덱스를 사용합니다. `index` 변수는 `tinyobj::index_t` 유형이며, `vertex_index`, `normal_index`, `texcoord_index` 멤버를 포함합니다. 이 인덱스를 사용하여 `attrib` 배열에서 실제 정점 속성을 찾아야 합니다:

```c++
vertex.pos = {
    attrib.vertices[3 * index.vertex_index + 0],
    attrib.vertices[3 * index.vertex_index + 1],
    attrib.vertices[3 * index.vertex_index + 2]
};

vertex.texCoord = {
    attrib.texcoords[2 * index.texcoord_index + 0],
    attrib.texcoords[2 * index.texcoord_index + 1]
};

vertex.color = {1.0f, 1.0f, 1.0f};
```

불행히도 `attrib.vertices` 배열은 `glm::vec3`와 같은 것이 아닌 `float` 값의 배열이므로 인덱스에 `3`을 곱해야 합니다. 텍스처 좌표의 경우 각 항목마다 두 개의 텍스처 좌표 구성 요소가 있습니다. `0`, `1`, `2`의 오프셋은 X, Y, Z 구성 요소 또는 텍스처 좌표의 경우 U와 V 구성 요소에 액세스하는 데 사용됩니다.

이제 최적화가 활성화된 상태로 프로그램을 실행하세요(예: Visual Studio의 `Release` 모드 및 GCC의 `-O3` 컴파일러 플래그). 그렇지 않으면 모델 로딩이 매우 느릴 것입니다. 다음과 같은 것을 볼 수 있어야 합니다:

![](/images/inverted_texture_coordinates.png)

훌륭합니다, 기하학은 정확해 보이지만 텍스처는 어떤가요? OBJ 형식은 수직 좌표 `0`이 이미지의 하단을 의미하는 좌표 체계를 가정합니다. 그러나 우리는 이미지를 Vulkan에 `0`이 이미지의 상단을 의미하는 상단에서 하단으로의 방향으로 업로드했습니다. 텍스처 좌표의 수직 구성 요소를 뒤집어 해결하세요:

```c++
vertex.texCoord = {
    attrib.texcoords[2 * index.texcoord_index + 0],
    1.0f - attrib.texcoords[2 * index.texcoord_index + 1]
};
```

이제 프로그램을 다시 실행하면 올바른 결과를 볼 수 있어야 합니다:

![](/images/drawing_model.png)

이렇게 힘든 작업이 이런 데모로 결실을 맺기 시작했습니다!

> 모델이 회전할 때 벽의 뒷면이 조금 이상하게 보일 수 있습니다. 이것은 정상이며, 단지 모델이 그 쪽에서 보기 위해 설계되지 않았기 때문입니다.

## 정점 중복 제거

불행히도 우리는 아직 인덱스 버퍼를 제대로 활용하고 있지 않습니다. `vertices` 벡터는 많은 중복된 정점 데이터를 포함하고 있습니다. 많은 정점들이 여러 삼각형에 포함되어 있기 때문입니다. 고유한 정점만 유지하고 나타날 때마다 인덱스 버퍼를 사용하여 재사용해야 합니다. 이를 구현하는 간단한 방법은 `map` 또는 `unordered_map`을 사용하여 고유한 정점과 해당 인덱스를 추적하는 것입니다:

```c++
#include <unordered_map>

...

std::unordered_map<Vertex, uint32_t> uniqueVertices{};

for (const auto& shape : shapes) {
    for (const auto& index : shape.mesh.indices) {
        Vertex vertex{};

        ...

        if (uniqueVertices.count(vertex) == 0) {
            uniqueVertices[vertex] = static_cast<uint32_t>(vertices.size());
            vertices.push_back(vertex);
        }

        indices.push_back(uniqueVertices[vertex]);
    }
}
```

OBJ 파일에서 정점을 읽을 때마다 이전에 동일한 위치와 텍스처 좌표를 가진 정점을 이미 본 적이 있는지 확인합니다. 그렇지 않은 경우 `vertices`에 추가하고 `uniqueVertices` 컨테이너에 인덱스를 저장합니다. 그 후 새 정점의 인덱스를 `indices`에 추가합니다. 이전에 정확히 동일한 정점을 본 경우 `uniqueVertices`에서 인덱스를 조회하고 그 인덱스를 `indices`에 저장합니다.

현재 프로그램은 컴파일에 실패할 것입니다. 왜냐하면 해시 테이블에서 사용자 정의 유형인 우리의 `Vertex` 구조체를 키로 사용하

려면 두 함수를 구현해야 하기 때문입니다: 등가성 테스트와 해시 계산입니다. 전자는 `Vertex` 구조체에서 `==` 연산자를 재정의함으로써 쉽게 구현할 수 있습니다:

```c++
bool operator==(const Vertex& other) const {
    return pos == other.pos && color == other.color && texCoord == other.texCoord;
}
```

`Vertex`에 대한 해시 함수는 `std::hash<T>`에 대한 템플릿 전문화를 지정함으로써 구현됩니다. 해시 함수는 복잡한 주제이지만, [cppreference.com은](http://en.cppreference.com/w/cpp/utility/hash) 구조체의 필드를 결합하여 괜찮은 품질의 해시 함수를 생성하는 다음과 같은 접근 방식을 권장합니다:

```c++
namespace std {
    template<> struct hash<Vertex> {
        size_t operator()(Vertex const& vertex) const {
            return ((hash<glm::vec3>()(vertex.pos) ^
                   (hash<glm::vec3>()(vertex.color) << 1)) >> 1) ^
                   (hash<glm::vec2>()(vertex.texCoord) << 1);
        }
    };
}
```

이 코드는 `Vertex` 구조체 외부에 배치되어야 합니다. GLM 유형에 대한 해시 함수는 다음 헤더를 사용하여 포함해야 합니다:

```c++
#define GLM_ENABLE_EXPERIMENTAL
#include <glm/gtx/hash.hpp>
```

해시 함수는 `gtx` 폴더에 정의되어 있으며, 이는 기술적으로 GLM에 대한 실험적 확장임을 의미합니다. 따라서 이를 사용하려면 `GLM_ENABLE_EXPERIMENTAL`을 정의해야 합니다. 이는 향후 GLM의 새 버전에서 API가 변경될 수 있음을 의미하지만, 실제로는 API가 매우 안정적입니다.

이제 프로그램을 성공적으로 컴파일하고 실행할 수 있어야 합니다. `vertices`의 크기를 확인하면 1,500,000에서 265,645로 줄어든 것을 볼 수 있습니다! 이는 평균적으로 각 정점이 약 6개의 삼각형에서 재사용된다는 것을 의미합니다. 이것은 확실히 많은 GPU 메모리를 절약합니다.

[C++ 코드](/code/28_model_loading.cpp) /
[버텍스 셰이더](/code/27_shader_depth.vert) /
[프래그먼트 셰이더](/code/27_shader_depth.frag)