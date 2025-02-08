# 개발 환경

이 챕터에서는 Vulkan 애플리케이션 개발을 위한 환경을 설정하고 몇 가지 유용한 라이브러리를 설치할 것입니다. 컴파일러를 제외한 우리가 사용할 모든 도구들은 Windows, Linux, MacOS와 호환되지만, 설치 단계가 조금씩 다르기 때문에 여기서는 별도로 설명합니다.

## Windows

Windows에서 개발하신다면, Visual Studio를 사용하여 코드를 컴파일한다고 가정하겠습니다. 완전한 C++17 지원을 위해서는 Visual Studio 2017이나 2019를 사용해야 합니다. 아래 설명된 단계들은 VS 2017을 기준으로 작성되었습니다.

### Vulkan SDK

Vulkan 애플리케이션을 개발하는 데 가장 중요한 구성 요소는 SDK입니다. SDK는 헤더, 표준 검증 계층, 디버깅 도구, 그리고 Vulkan 함수를 위한 로더를 포함합니다. 이 로더는 OpenGL의 GLEW와 비슷하게 - 익숙하시다면 - 런타임에 드라이버에서 함수들을 찾습니다.

SDK는 [LunarG 웹사이트](https://vulkan.lunarg.com/)에서 페이지 하단의 버튼을 사용하여 다운로드할 수 있습니다. 계정을 만들 필요는 없지만, 계정이 있으면 유용할 수 있는 추가 문서에 접근할 수 있습니다.

![](/images/vulkan_sdk_download_buttons.png)

설치를 진행하면서 SDK의 설치 위치에 주의를 기울이세요. 가장 먼저 할 일은 그래픽 카드와 드라이버가 Vulkan을 제대로 지원하는지 확인하는 것입니다. SDK를 설치한 디렉토리로 가서 `Bin` 디렉토리를 열고 `vkcube.exe` 데모를 실행하세요. 다음과 같은 화면이 보여야 합니다:

![](/images/cube_demo.png)

오류 메시지가 나타난다면 드라이버가 최신 상태인지, Vulkan 런타임을 포함하고 있는지, 그리고 그래픽 카드가 지원되는지 확인하세요. 주요 벤더의 드라이버 링크는 [소개 챕터](!ko/Introduction)를 참조하세요.

이 디렉토리에는 개발에 유용한 또 다른 프로그램이 있습니다. `glslangValidator.exe`와 `glslc.exe` 프로그램은 사람이 읽을 수 있는 [GLSL](https://en.wikipedia.org/wiki/OpenGL_Shading_Language)을 바이트코드로 컴파일하는 데 사용됩니다. 이에 대해서는 [셰이더 모듈](!ko/Drawing_a_triangle/Graphics_pipeline_basics/Shader_modules) 챕터에서 자세히 다룰 것입니다. `Bin` 디렉토리에는 또한 Vulkan 로더와 검증 계층의 바이너리가 들어있고, `Lib` 디렉토리에는 라이브러리가 들어있습니다.

마지막으로, Vulkan 헤더가 포함된 `Include` 디렉토리가 있습니다. 다른 파일들도 자유롭게 살펴보셔도 되지만, 이 튜토리얼에서는 필요하지 않을 것입니다.

### GLFW

앞서 언급했듯이, Vulkan 자체는 플랫폼에 구애받지 않는 API이며 렌더링된 결과를 표시할 윈도우를 생성하는 도구를 포함하지 않습니다. Vulkan의 크로스 플랫폼 이점을 활용하고 Win32의 복잡함을 피하기 위해, Windows, Linux, MacOS를 지원하는 [GLFW 라이브러리](http://www.glfw.org/)를 사용하여 윈도우를 생성할 것입니다. 이 목적으로 [SDL](https://www.libsdl.org/) 같은 다른 라이브러리들도 있지만, GLFW의 장점은 윈도우 생성 외에도 Vulkan의 다른 플랫폼 특정적인 부분들도 추상화한다는 것입니다.

GLFW의 최신 릴리스는 [공식 웹사이트](http://www.glfw.org/download.html)에서 찾을 수 있습니다. 이 튜토리얼에서는 64비트 바이너리를 사용할 것이지만, 물론 32비트 모드로 빌드하는 것을 선택할 수도 있습니다. 그 경우 `Lib` 대신 `Lib32` 디렉토리에 있는 Vulkan SDK 바이너리와 링크해야 합니다. 다운로드 후, 아카이브를 편리한 위치에 압축 해제하세요. 저는 문서의 Visual Studio 디렉토리 아래에 `Libraries` 디렉토리를 만들기로 했습니다.

![](/images/glfw_directory.png)

### GLM

DirectX 12와 달리, Vulkan은 선형 대수 연산을 위한 라이브러리를 포함하지 않으므로 하나를 다운로드해야 합니다. [GLM](http://glm.g-truc.net/)은 그래픽스 API와 함께 사용하도록 설계된 좋은 라이브러리이며 OpenGL에서도 일반적으로 사용됩니다.

GLM은 헤더 전용 라이브러리이므로 [최신 버전](https://github.com/g-truc/glm/releases)을 다운로드하여 편리한 위치에 저장하기만 하면 됩니다. 이제 다음과 같은 디렉토리 구조를 가지게 될 것입니다:

![](/images/library_directory.png)

### Visual Studio 설정

이제 모든 의존성을 설치했으므로 Vulkan을 위한 기본 Visual Studio 프로젝트를 설정하고 모든 것이 제대로 작동하는지 확인하기 위해 약간의 코드를 작성할 수 있습니다.

Visual Studio를 시작하고 이름을 입력하고 `확인`을 눌러 새로운 `Windows Desktop Wizard` 프로젝트를 만드세요.

![](/images/vs_new_cpp_project.png)

디버그 메시지를 출력할 곳이 있도록 애플리케이션 유형으로 `Console Application (.exe)`가 선택되어 있는지 확인하고, Visual Studio가 상용구 코드를 추가하지 않도록 `Empty Project`를 체크하세요.

![](/images/vs_application_settings.png)

`확인`을 눌러 프로젝트를 만들고 C++ 소스 파일을 추가하세요. 이미 이 방법을 알고 계시겠지만, 완전성을 위해 단계들이 여기 포함되어 있습니다.

![](/images/vs_new_item.png)

![](/images/vs_new_source_file.png)

이제 다음 코드를 파일에 추가하세요. 지금 당장 이해하려고 하지 마세요; 우리는 단지 Vulkan 애플리케이션을 컴파일하고 실행할 수 있는지 확인하고 있는 것입니다. 다음 챕터에서 처음부터 시작할 것입니다.

```c++
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>

#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEPTH_ZERO_TO_ONE
#include <glm/vec4.hpp>
#include <glm/mat4x4.hpp>

#include <iostream>

int main() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    GLFWwindow* window = glfwCreateWindow(800, 600, "Vulkan window", nullptr, nullptr);

    uint32_t extensionCount = 0;
    vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);

    std::cout << extensionCount << " extensions supported\n";

    glm::mat4 matrix;
    glm::vec4 vec;
    auto test = matrix * vec;

    while(!glfwWindowShouldClose(window)) {
        glfwPollEvents();
    }

    glfwDestroyWindow(window);

    glfwTerminate();

    return 0;
}
```

이제 오류를 제거하기 위해 프로젝트를 구성해 보겠습니다. 프로젝트 속성 대화 상자를 열고 대부분의 설정이 `Debug`와 `Release` 모드 모두에 적용되므로 `All Configurations`가 선택되어 있는지 확인하세요.

![](/images/vs_open_project_properties.png)

![](/images/vs_all_configs.png)

`C++ -> General -> Additional Include Directories`로 가서 드롭다운 박스에서 `<Edit...>`를 누르세요.

![](/images/vs_cpp_general.png)

Vulkan, GLFW, GLM의 헤더 디렉토리를 추가하세요:

![](/images/vs_include_dirs.png)

다음으로, `Linker -> General`에서 라이브러리 디렉토리 편집기를 여세요:

![](/images/vs_link_settings.png)

그리고 Vulkan과 GLFW의 오브젝트 파일 위치를 추가하세요:

![](/images/vs_link_dirs.png)

`Linker -> Input`으로 가서 `Additional Dependencies` 드롭다운 박스에서 `<Edit...>`를 누르세요.

![](/images/vs_link_input.png)

Vulkan과 GLFW 오브젝트 파일의 이름을 입력하세요:

![](/images/vs_dependencies.png)

그리고 마지막으로 C++17 기능을 지원하도록 컴파일러를 변경하세요:

![](/images/vs_cpp17.png)

이제 프로젝트 속성 대화 상자를 닫을 수 있습니다. 모든 것을 올바르게 했다면 코드에서 더 이상 오류가 강조 표시되지 않아야 합니다.

마지막으로, 실제로 64비트 모드에서 컴파일하고 있는지 확인하세요:

![](/images/vs_build_mode.png)

F5를 눌러 프로젝트를 컴파일하고 실행하면 다음과 같이 명령 프롬프트와 창이 나타나야 합니다:

![](/images/vs_test_window.png)

확장 개수가 0이 아니어야 합니다. 축하합니다, 이제 [Vulkan과 놀아볼](!ko/Drawing_a_triangle/Setup/Base_code) 준비가 되었습니다!

## Linux

이 지침들은 Ubuntu, Fedora, Arch Linux 사용자를 대상으로 하지만, 패키지 관리자별 명령을 자신에게 맞는 것으로 변경하여 따라할 수 있을 것입니다. C++17을 지원하는 컴파일러(GCC 7+ 또는 Clang 5+)가 있어야 합니다. 또한 `make`도 필요합니다.

### Vulkan 패키지

Linux에서 Vulkan 애플리케이션을 개발하는 데 가장 중요한 구성 요소는 Vulkan 로더, 검증 계층, 그리고 시스템이 Vulkan을 지원하는지 테스트하기 위한 몇 가지 명령줄 유틸리티입니다:

* `sudo apt install vulkan-tools` 또는 `sudo dnf install vulkan-tools`: 명령줄 유틸리티, 가장 중요한 것은 `vulkaninfo`와 `vkcube`입니다. 이것들을 실행하여 시스템이 Vulkan을 지원하는지 확인하세요.
* `sudo apt install libvulkan-dev` 또는 `sudo dnf install vulkan-loader-devel`: Vulkan 로더를 설치합니다. 이 로더는 OpenGL의 GLEW와 비슷하게 - 익숙하시다면 - 런타임에 드라이버에서 함수들을 찾습니다.
* `sudo apt install vulkan-validationlayers spirv-tools` 또는 `sudo dnf install mesa-vulkan-devel vulkan-validation-layers-devel`: 표준 검증 계층과 필요한 SPIR-V 도구를 설치합니다. 이것들은 Vulkan 애플리케이션을 디버깅할 때 매우 중요하며, 다음 챕터에서 설명할 것입니다.

Arch Linux에서는 `sudo pacman -S vulkan-devel`을 실행하여 위의 모든 필요한 도구를 설치할 수 있습니다.

설치가 성공적이었다면, Vulkan 부분은 모두 준비된 것입니다. `vkcube`를 실행하여 다음과 같은 창이 나타나는지 확인하세요:

![](/images/cube_demo_nowindow.png)

오류 메시지가 나타난다면 드라이버가 최신 상태인지, Vulkan 런타임을 포함하고 있는지, 그리고 그래픽 카드가 지원되는지 확인하세요. 주요 벤더의 드라이버 링크는 [소개 챕터](!ko/Introduction)를 참조하세요.

### X Window System과 XFree86-VidModeExtension
시스템에 이 라이브러리들이 없을 수 있습니다. 없다면 다음 명령어로 설치할 수 있습니다:
* `sudo apt install libxxf86vm-dev` 또는 `dnf install libXxf86vm-devel`: XFree86-VidModeExtension에 대한 인터페이스를 제공합니다.
* `sudo apt install libxi-dev` 또는 `dnf install libXi-devel`: XINPUT 확장에 대한 X Window System 클라이언트 인터페이스를 제공합니다.

### GLFW

앞서 언급했듯이, Vulkan 자체는 플랫폼에 구애받지 않는 API이며 렌더링된 결과를 표시할 윈도우를 생성하는 도구를 포함하지 않습니다. Vulkan의 크로스 플랫폼 이점을 활용하고 X11의 복잡함을 피하기 위해, Windows, Linux, MacOS를 지원하는 [GLFW 라이브러리](http://www.glfw.org/)를 사용하여 윈도우를 생성할 것입니다. 이 목적으로 [SDL](https://www.libsdl.org/) 같은 다른 라이브러리들도 있지만, GLFW의 장점은 윈도우 생성 외에도 Vulkan의 다른 플랫폼 특정적인 부분들도 추상화한다는 것입니다.

다음 명령어로 GLFW를 설치할 것입니다:

```bash
sudo apt install libglfw3-dev
```
또는
```bash
sudo dnf install glfw-devel
```
또는
```bash
sudo pacman -S glfw
```

### GLM

DirectX 12와 달리, Vulkan은 선형 대수 연산을 위한 라이브러리를 포함하지 않으므로 하나를 다운로드해야 합니다. [GLM](http://glm.g-truc.net/)은 그래픽스 API와 함께 사용하도록 설계된 좋은 라이브러리이며 OpenGL에서도 일반적으로 사용됩니다.

이것은 `libglm-dev` 또는 `glm-devel` 패키지에서 설치할 수 있는 헤더 전용 라이브러리입니다:

```bash
sudo apt install libglm-dev
```
또는
```bash
sudo dnf install glm-devel
```
또는
```bash
sudo pacman -S glm
```

### 셰이더 컴파일러

거의 모든 것이 준비되었지만, 사람이 읽을 수 있는 [GLSL](https://en.wikipedia.org/wiki/OpenGL_Shading_Language)을 바이트코드로 컴파일하는 프로그램이 필요합니다.

Khronos Group의 `glslangValidator`와 Google의 `glslc`, 이 두 가지가 인기 있는 셰이더 컴파일러입니다. 후자는 GCC와 Clang과 비슷한 사용법을 가지고 있어서 우리는 이것을 선택할 것입니다: Ubuntu에서는 Google의 [비공식 바이너리](https://github.com/google/shaderc/blob/main/downloads.md)를 다운로드하고 `glslc`를 `/usr/local/bin`에 복사하세요. 권한에 따라 `sudo`가 필요할 수 있습니다. Fedora에서는 `sudo dnf install glslc`를, Arch Linux에서는 `sudo pacman -S shaderc`를 실행하세요. 테스트를 위해 `glslc`를 실행하면 컴파일할 셰이더를 전달하지 않았다고 정당하게 불평해야 합니다:

`glslc: error: no input files`

`glslc`에 대해서는 [셰이더 모듈](!ko/Drawing_a_triangle/Graphics_pipeline_basics/Shader_modules) 챕터에서 자세히 다룰 것입니다.

### 메이크파일 프로젝트 설정하기

이제 모든 의존성을 설치했으므로 Vulkan을 위한 기본 메이크파일 프로젝트를 설정하고 모든 것이 제대로 작동하는지 확인하기 위해 약간의 코드를 작성할 수 있습니다.

`VulkanTest`와 같은 이름으로 편리한 위치에 새 디렉토리를 만드세요. `main.cpp`라는 소스 파일을 만들고 다음 코드를 삽입하세요. 지금 당장 이해하려고 하지 마세요; 우리는 단지 Vulkan 애플리케이션을 컴파일하고 실행할 수 있는지 확인하고 있는 것입니다. 다음 챕터에서 처음부터 시작할 것입니다.

```c++
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>

#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEPTH_ZERO_TO_ONE
#include <glm/vec4.hpp>
#include <glm/mat4x4.hpp>

#include <iostream>

int main() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    GLFWwindow* window = glfwCreateWindow(800, 600, "Vulkan window", nullptr, nullptr);

    uint32_t extensionCount = 0;
    vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);

    std::cout << extensionCount << " extensions supported\n";

    glm::mat4 matrix;
    glm::vec4 vec;
    auto test = matrix * vec;

    while(!glfwWindowShouldClose(window)) {
        glfwPollEvents();
    }

    glfwDestroyWindow(window);

    glfwTerminate();

    return 0;
}
```

다음으로, 이 기본적인 Vulkan 코드를 컴파일하고 실행하기 위한 메이크파일을 작성해보겠습니다. `Makefile`이라는 이름의 새 빈 파일을 만드세요. 변수와 규칙이 어떻게 작동하는지와 같은 메이크파일에 대한 기본적인 경험이 있다고 가정하겠습니다. 그렇지 않다면, [이 튜토리얼](https://makefiletutorial.com/)로 빠르게 배울 수 있습니다.

먼저 파일의 나머지 부분을 단순화하기 위해 몇 가지 변수를 정의하겠습니다.
기본 컴파일러 플래그를 지정할 `CFLAGS` 변수를 정의하세요:

```make
CFLAGS = -std=c++17 -O2
```

우리는 현대적인 C++(`-std=c++17`)를 사용할 것이며, 최적화 레벨을 O2로 설정할 것입니다. 프로그램을 더 빨리 컴파일하기 위해 -O2를 제거할 수 있지만, 릴리스 빌드에서는 다시 넣는 것을 잊지 말아야 합니다.

비슷하게, 링커 플래그를 `LDFLAGS` 변수에 정의하세요:

```make
LDFLAGS = -lglfw -lvulkan -ldl -lpthread -lX11 -lXxf86vm -lXrandr -lXi
```

`-lglfw` 플래그는 GLFW용이고, `-lvulkan`은 Vulkan 함수 로더와 링크하며, 나머지 플래그들은 GLFW가 필요로 하는 저수준 시스템 라이브러리입니다. 나머지 플래그들은 GLFW 자체의 의존성입니다: 스레딩과 윈도우 관리를 위한 것입니다.

`Xxf86vm`과 `Xi` 라이브러리가 아직 시스템에 설치되어 있지 않을 수 있습니다. 다음 패키지들에서 찾을 수 있습니다:

```bash
sudo apt install libxxf86vm-dev libxi-dev
```
또는
```bash
sudo dnf install libXi-devel libXxf86vm-devel
```
또는
```bash
sudo pacman -S libxi libxxf86vm
```

이제 `VulkanTest`를 컴파일하는 규칙을 지정하는 것은 간단합니다. 들여쓰기에 공백 대신 탭을 사용해야 합니다.

```make
VulkanTest: main.cpp
    g++ $(CFLAGS) -o VulkanTest main.cpp $(LDFLAGS)
```

메이크파일을 저장하고 `main.cpp`와 `Makefile`이 있는 디렉토리에서 `make`를 실행하여 이 규칙이 작동하는지 확인하세요. 이렇게 하면 `VulkanTest` 실행 파일이 생성되어야 합니다.

이제 두 가지 규칙을 더 정의하겠습니다. `test`는 실행 파일을 실행하고 `clean`은 빌드된 실행 파일을 제거합니다:

```make
.PHONY: test clean

test: VulkanTest
    ./VulkanTest

clean:
    rm -f VulkanTest
```

`make test`를 실행하면 프로그램이 성공적으로 실행되고 Vulkan 확장 개수가 표시되어야 합니다. 빈 창을 닫으면 애플리케이션이 성공 반환 코드(`0`)와 함께 종료되어야 합니다. 이제 다음과 같은 완전한 메이크파일이 있어야 합니다:

```make
CFLAGS = -std=c++17 -O2
LDFLAGS = -lglfw -lvulkan -ldl -lpthread -lX11 -lXxf86vm -lXrandr -lXi

VulkanTest: main.cpp
    g++ $(CFLAGS) -o VulkanTest main.cpp $(LDFLAGS)

.PHONY: test clean

test: VulkanTest
    ./VulkanTest

clean:
    rm -f VulkanTest
```

이제 이 디렉토리를 Vulkan 프로젝트의 템플릿으로 사용할 수 있습니다. 복사본을 만들고 `HelloTriangle`과 같은 이름으로 변경한 다음 `main.cpp`의 모든 코드를 제거하세요.

이제 [진짜 모험](!ko/Drawing_a_triangle/Setup/Base_code)을 시작할 준비가 되었습니다.

## MacOS

이 지침은 Xcode와 [Homebrew 패키지 관리자](https://brew.sh/)를 사용한다고 가정합니다. 또한, MacOS 버전 10.11 이상이 필요하고 기기가 [Metal API](https://en.wikipedia.org/wiki/Metal_(API)#Supported_GPUs)를 지원해야 한다는 점을 기억하세요.

### Vulkan SDK

Vulkan 애플리케이션을 개발하는 데 가장 중요한 구성 요소는 SDK입니다. SDK는 헤더, 표준 검증 계층, 디버깅 도구, 그리고 Vulkan 함수를 위한 로더를 포함합니다. 이 로더는 OpenGL의 GLEW와 비슷하게 - 익숙하시다면 - 런타임에 드라이버에서 함수들을 찾습니다.

SDK는 [LunarG 웹사이트](https://vulkan.lunarg.com/)에서 페이지 하단의 버튼을 사용하여 다운로드할 수 있습니다. 계정을 만들 필요는 없지만, 계정이 있으면 유용할 수 있는 추가 문서에 접근할 수 있습니다.

![](/images/vulkan_sdk_download_buttons.png)

MacOS용 SDK 버전은 내부적으로 [MoltenVK](https://moltengl.com/)를 사용합니다. MacOS에는 Vulkan에 대한 네이티브 지원이 없으므로, MoltenVK는 실제로 Vulkan API 호출을 Apple의 Metal 그래픽스 프레임워크로 변환하는 계층 역할을 합니다. 이를 통해 Apple의 Metal 프레임워크의 디버깅과 성능 이점을 활용할 수 있습니다.

다운로드 후, 내용을 원하는 폴더에 추출하기만 하면 됩니다(Xcode에서 프로젝트를 만들 때 참조해야 한다는 점을 기억하세요). 추출된 폴더 안의 `Applications` 폴더에 SDK를 사용하여 몇 가지 데모를 실행할 수 있는 실행 파일들이 있어야 합니다. `vkcube` 실행 파일을 실행하면 다음과 같은 화면이 나타날 것입니다:

![](/images/cube_demo_mac.png)

### GLFW

앞서 언급했듯이, Vulkan 자체는 플랫폼에 구애받지 않는 API이며 렌더링된 결과를 표시할 윈도우를 생성하는 도구를 포함하지 않습니다. Windows, Linux, MacOS를 지원하는 [GLFW 라이브러리](http://www.glfw.org/)를 사용하여 윈도우를 생성할 것입니다. 이 목적으로 [SDL](https://www.libsdl.org/) 같은 다른 라이브러리들도 있지만, GLFW의 장점은 윈도우 생성 외에도 Vulkan의 다른 플랫폼 특정적인 부분들도 추상화한다는 것입니다.

MacOS에 GLFW를 설치하기 위해 Homebrew 패키지 관리자를 사용하여 `glfw` 패키지를 설치할 것입니다:

```bash
brew install glfw
```

### GLM

Vulkan은 선형 대수 연산을 위한 라이브러리를 포함하지 않으므로 하나를 다운로드해야 합니다. [GLM](http://glm.g-truc.net/)은 그래픽스 API와 함께 사용하도록 설계된 좋은 라이브러리이며 OpenGL에서도 일반적으로 사용됩니다.

이것은 `glm` 패키지에서 설치할 수 있는 헤더 전용 라이브러리입니다:

```bash
brew install glm
```

### Xcode 설정하기

이제 모든 의존성이 설치되었으므로 Vulkan을 위한 기본 Xcode 프로젝트를 설정할 수 있습니다. 여기서의 대부분의 지침은 본질적으로 모든 의존성을 프로젝트에 연결하기 위한 많은 "배관 작업"입니다. 또한 다음 지침에서 `vulkansdk` 폴더를 언급할 때마다 Vulkan SDK를 추출한 폴더를 참조하고 있다는 점을 기억하세요.

Xcode를 시작하고 새 Xcode 프로젝트를 만드세요. 열리는 창에서 Application > Command Line Tool을 선택하세요.

![](/images/xcode_new_project.png)

`Next`를 선택하고, 프로젝트 이름을 입력하고 `Language`에서 `C++`를 선택하세요.

![](/images/xcode_new_project_2.png)

`Next`를 누르면 프로젝트가 생성되었을 것입니다. 이제 생성된 `main.cpp` 파일의 코드를 다음 코드로 변경해 보겠습니다:

```c++
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>

#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEPTH_ZERO_TO_ONE
#include <glm/vec4.hpp>
#include <glm/mat4x4.hpp>

#include <iostream>

int main() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    GLFWwindow* window = glfwCreateWindow(800, 600, "Vulkan window", nullptr, nullptr);

    uint32_t extensionCount = 0;
    vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);

    std::cout << extensionCount << " extensions supported\n";

    glm::mat4 matrix;
    glm::vec4 vec;
    auto test = matrix * vec;

    while(!glfwWindowShouldClose(window)) {
        glfwPollEvents();
    }

    glfwDestroyWindow(window);

    glfwTerminate();

    return 0;
}
```

이 모든 코드가 하는 일을 아직 이해할 필요는 없습니다. 우리는 단지 모든 것이 제대로 작동하는지 확인하기 위해 몇 가지 API 호출을 설정하고 있을 뿐입니다.

Xcode는 이미 찾을 수 없는 라이브러리와 같은 몇 가지 오류를 표시하고 있을 것입니다. 이제 이러한 오류들을 제거하기 위해 프로젝트를 구성하기 시작하겠습니다. *Project Navigator* 패널에서 프로젝트를 선택하세요. *Build Settings* 탭을 열고:

* **Header Search Paths** 필드를 찾아 `/usr/local/include` 링크를 추가하세요(Homebrew가 헤더를 설치하는 곳이므로 glm과 glfw3 헤더 파일이 여기에 있어야 합니다)와 Vulkan 헤더를 위한 `vulkansdk/macOS/include` 링크를 추가하세요.
* **Library Search Paths** 필드를 찾아 `/usr/local/lib` 링크를 추가하세요(마찬가지로 Homebrew가 라이브러리를 설치하는 곳이므로 glm과 glfw3 라이브러리 파일이 여기에 있어야 합니다)와 `vulkansdk/macOS/lib` 링크를 추가하세요.

다음과 같이 보여야 합니다(당연히 파일을 어디에 두었는지에 따라 경로가 다를 수 있습니다):

![](/images/xcode_paths.png)

이제 *Build Phases* 탭의 **Link Binary With Libraries**에서 `glfw3`와 `vulkan` 프레임워크를 모두 추가할 것입니다. 작업을 쉽게 하기 위해 프로젝트에 동적 라이브러리를 추가할 것입니다(정적 프레임워크를 사용하고 싶다면 이들 라이브러리의 문서를 확인하세요).

* glfw의 경우 `/usr/local/lib` 폴더를 열면 `libglfw.3.x.dylib`와 같은 이름의 파일이 있을 것입니다("x"는 라이브러리의 버전 번호로, Homebrew에서 패키지를 다운로드한 시기에 따라 다를 수 있습니다). 해당 파일을 Xcode의 Linked Frameworks and Libraries 탭으로 드래그하기만 하면 됩니다.
* vulkan의 경우, `vulkansdk/macOS/lib`로 이동하세요. `libvulkan.1.dylib`와 `libvulkan.1.x.xx.dylib` 파일 모두에 대해 같은 작업을 하세요("x"는 다운로드한 SDK의 버전 번호입니다).

이러한 라이브러리들을 추가한 후, 같은 탭의 **Copy Files**에서 `Destination`을 "Frameworks"로 변경하고, 하위 경로를 지우고 "Copy only when installing"의 선택을 해제하세요. "+" 기호를 클릭하고 이 세 프레임워크를 여기에도 모두 추가하세요.

Xcode 구성이 다음과 같이 보여야 합니다:

![](/images/xcode_frameworks.png)

마지막으로 설정해야 할 것은 몇 가지 환경 변수입니다. Xcode 툴바에서 `Product` > `Scheme` > `Edit Scheme...`로 이동하여 `Arguments` 탭에서 다음 두 환경 변수를 추가하세요:

* VK_ICD_FILENAMES = `vulkansdk/macOS/share/vulkan/icd.d/MoltenVK_icd.json`
* VK_LAYER_PATH = `vulkansdk/macOS/share/vulkan/explicit_layer.d`

다음과 같이 보여야 합니다:

![](/images/xcode_variables.png)

마지막으로, 모든 설정이 완료되었습니다! 이제 프로젝트를 실행하면(선택한 구성에 따라 빌드 구성을 Debug 또는 Release로 설정하는 것을 잊지 마세요) 다음과 같은 화면이 보일 것입니다:

![](/images/xcode_output.png)

확장 개수가 0이 아니어야 합니다. 다른 로그들은 라이브러리들에서 나온 것으로, 구성에 따라 다른 메시지가 표시될 수 있습니다.

이제 [진짜 시작](!ko/Drawing_a_triangle/Setup/Base_code)을 할 준비가 되었습니다.