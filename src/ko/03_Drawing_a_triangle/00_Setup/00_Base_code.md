# 기본 코드

## 기본 구조

이전 장에서는 적절한 구성으로 Vulkan 프로젝트를 만들고 샘플 코드로 테스트해보았습니다. 이번 장에서는 다음 코드부터 시작하겠습니다:

```c++
#include <vulkan/vulkan.h>

#include <iostream>
#include <stdexcept>
#include <cstdlib>

class HelloTriangleApplication {
public:
    void run() {
        initVulkan();
        mainLoop();
        cleanup();
    }

private:
    void initVulkan() {

    }

    void mainLoop() {

    }

    void cleanup() {

    }
};

int main() {
    HelloTriangleApplication app;

    try {
        app.run();
    } catch (const std::exception& e) {
        std::cerr << e.what() << std::endl;
        return EXIT_FAILURE;
    }

    return EXIT_SUCCESS;
}
```

먼저 LunarG SDK의 Vulkan 헤더를 포함시킵니다. 이 헤더는 함수, 구조체, 열거형을 제공합니다. `stdexcept`와 `iostream` 헤더는 오류를 보고하고 전파하는 데 사용됩니다. `cstdlib` 헤더는 `EXIT_SUCCESS`와 `EXIT_FAILURE` 매크로를 제공합니다.

프로그램은 클래스로 감싸져 있으며, 여기에 Vulkan 객체들을 private 클래스 멤버로 저장하고 각각을 초기화하는 함수들을 추가할 것입니다. 이 함수들은 `initVulkan` 함수에서 호출됩니다. 모든 준비가 완료되면 프레임 렌더링을 시작하기 위해 메인 루프로 진입합니다. 잠시 후에 `mainLoop` 함수에 창이 닫힐 때까지 반복하는 루프를 추가할 것입니다. 창이 닫히고 `mainLoop`가 반환되면, `cleanup` 함수에서 사용했던 리소스들을 해제할 것입니다.

실행 중에 치명적인 오류가 발생하면 설명이 포함된 `std::runtime_error` 예외를 던집니다. 이 예외는 `main` 함수로 전파되어 명령 프롬프트에 출력됩니다. 다양한 표준 예외 타입을 처리하기 위해 더 일반적인 `std::exception`을 catch합니다. 곧 다루게 될 오류의 한 예시는 필요한 확장 기능이 지원되지 않는 경우입니다.

이 장 이후에 나오는 모든 장은 `initVulkan`에서 호출될 새로운 함수 하나와 `cleanup`에서 마지막에 해제해야 할 하나 이상의 새로운 Vulkan 객체를 클래스의 private 멤버에 추가할 것입니다.

## 리소스 관리

`malloc`으로 할당된 모든 메모리가 `free` 호출을 필요로 하는 것처럼, 우리가 생성하는 모든 Vulkan 객체는 더 이상 필요하지 않을 때 명시적으로 파괴되어야 합니다. C++에서는 [RAII](https://en.wikipedia.org/wiki/Resource_Acquisition_Is_Initialization)나 `<memory>` 헤더에서 제공하는 스마트 포인터를 사용하여 자동 리소스 관리가 가능합니다. 하지만 이 튜토리얼에서는 Vulkan 객체의 할당과 해제를 명시적으로 하기로 했습니다. 결국 Vulkan의 특징은 실수를 방지하기 위해 모든 작업을 명시적으로 하는 것이므로, API가 어떻게 작동하는지 배우기 위해서는 객체의 수명을 명시적으로 다루는 것이 좋습니다.

이 튜토리얼을 따라한 후에는 생성자에서 Vulkan 객체를 획득하고 소멸자에서 해제하는 C++ 클래스를 작성하거나, 소유권 요구사항에 따라 `std::unique_ptr` 또는 `std_shared_ptr`에 커스텀 삭제자를 제공하여 자동 리소스 관리를 구현할 수 있습니다. RAII는 더 큰 Vulkan 프로그램에 권장되는 모델이지만, 학습 목적으로는 뒤에서 어떤 일이 일어나는지 아는 것이 좋습니다.

Vulkan 객체는 `vkCreateXXX`와 같은 함수로 직접 생성되거나, `vkAllocateXXX`와 같은 함수로 다른 객체를 통해 할당됩니다. 객체가 더 이상 어디에서도 사용되지 않는다는 것을 확인한 후에는 `vkDestroyXXX`와 `vkFreeXXX` 같은 대응되는 함수로 파괴해야 합니다. 이러한 함수들의 매개변수는 객체 타입마다 다르지만, 모두가 공유하는 하나의 매개변수가 있습니다: `pAllocator`. 이는 커스텀 메모리 할당자를 위한 콜백을 지정할 수 있는 선택적 매개변수입니다. 이 튜토리얼에서는 이 매개변수를 무시하고 항상 `nullptr`을 인자로 전달할 것입니다.

## GLFW 통합하기

오프스크린 렌더링을 위해 Vulkan을 사용하려는 경우 창을 만들지 않아도 완벽하게 작동하지만, 실제로 무언가를 보여주는 것이 훨씬 더 흥미롭죠! 먼저 `#include <vulkan/vulkan.h>` 라인을 다음으로 교체하세요:

```c++
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>
```

이렇게 하면 GLFW가 자체 정의를 포함하고 자동으로 Vulkan 헤더를 로드합니다. `initWindow` 함수를 추가하고 다른 호출들 전에 `run` 함수에서 이를 호출하도록 추가하세요. 이 함수를 사용하여 GLFW를 초기화하고 창을 만들 것입니다.

```c++
void run() {
    initWindow();
    initVulkan();
    mainLoop();
    cleanup();
}

private:
    void initWindow() {

    }
```

`initWindow`의 첫 번째 호출은 GLFW 라이브러리를 초기화하는 `glfwInit()`이어야 합니다. GLFW는 원래 OpenGL 컨텍스트를 만들기 위해 설계되었기 때문에, 다음 호출로 OpenGL 컨텍스트를 만들지 않도록 지시해야 합니다:

```c++
glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
```

창 크기 조정은 나중에 살펴볼 특별한 처리가 필요하므로, 지금은 다음과 같은 창 힌트 호출로 비활성화하세요:

```c++
glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);
```

이제 실제 창을 만드는 일만 남았습니다. 참조를 저장하기 위해 `GLFWwindow* window;` private 클래스 멤버를 추가하고 다음과 같이 창을 초기화하세요:

```c++
window = glfwCreateWindow(800, 600, "Vulkan", nullptr, nullptr);
```

처음 세 매개변수는 창의 너비, 높이, 제목을 지정합니다. 네 번째 매개변수는 선택적으로 창을 열 모니터를 지정할 수 있고, 마지막 매개변수는 OpenGL에만 관련이 있습니다.

앞으로 이 값들을 여러 번 참조할 것이므로 하드코딩된 너비와 높이 숫자 대신 상수를 사용하는 것이 좋습니다. `HelloTriangleApplication` 클래스 정의 위에 다음 라인들을 추가했습니다:

```c++
const uint32_t WIDTH = 800;
const uint32_t HEIGHT = 600;
```

그리고 창 생성 호출을 다음과 같이 교체했습니다:

```c++
window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
```

이제 `initWindow` 함수가 다음과 같이 보일 것입니다:

```c++
void initWindow() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);

    window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
}
```

오류가 발생하거나 창이 닫힐 때까지 애플리케이션을 실행 상태로 유지하기 위해, `mainLoop` 함수에 다음과 같은 이벤트 루프를 추가해야 합니다:

```c++
void mainLoop() {
    while (!glfwWindowShouldClose(window)) {
        glfwPollEvents();
    }
}
```

이 코드는 꽤 자명합니다. 사용자가 창을 닫을 때까지 루프를 돌며 X 버튼을 누르는 것과 같은 이벤트를 확인합니다. 이는 또한 나중에 단일 프레임을 렌더링하는 함수를 호출할 루프이기도 합니다.

창이 닫히면 창을 파괴하고 GLFW 자체를 종료하여 리소스를 정리해야 합니다. 이것이 우리의 첫 번째 `cleanup` 코드가 될 것입니다:

```c++
void cleanup() {
    glfwDestroyWindow(window);

    glfwTerminate();
}
```

이제 프로그램을 실행하면 창을 닫아서 애플리케이션이 종료될 때까지 `Vulkan`이라는 제목의 창이 표시되는 것을 볼 수 있습니다. 이제 Vulkan 애플리케이션의 뼈대가 마련되었으니, [첫 번째 Vulkan 객체를 만들어봅시다](!en/Drawing_a_triangle/Setup/Instance)!

[C++ 코드](/code/00_base_code.cpp)