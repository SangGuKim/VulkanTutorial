# 윈도우 서피스

Vulkan은 플랫폼에 독립적인 API이므로, 직접 윈도우 시스템과 인터페이스할 수 없습니다. Vulkan과 윈도우 시스템 간의 연결을 설정하여 화면에 결과를 표시하기 위해서는 WSI(Window System Integration) extension을 사용해야 합니다. 이 장에서는 첫 번째로 `VK_KHR_surface`를 다룰 것입니다. 이는 렌더링된 이미지를 표시할 추상 타입의 서피스를 나타내는 `VkSurfaceKHR` 객체를 제공합니다. 우리 프로그램의 서피스는 GLFW로 이미 열어둔 윈도우가 뒷받침할 것입니다.

`VK_KHR_surface` extension은 인스턴스 레벨 extension이며, 이미 활성화되어 있습니다. `glfwGetRequiredInstanceExtensions`가 반환하는 목록에 포함되어 있기 때문입니다. 이 목록에는 다음 몇 장에서 사용할 다른 WSI extension들도 포함되어 있습니다.

윈도우 서피스는 인스턴스 생성 직후에 생성되어야 합니다. 물리 장치 선택에 영향을 미칠 수 있기 때문입니다. 이를 미룬 이유는 윈도우 서피스가 렌더 타겟과 프레젠테이션이라는 더 큰 주제의 일부이며, 이에 대한 설명이 기본 설정을 복잡하게 만들었을 것이기 때문입니다. 또한 윈도우 서피스는 Vulkan에서 완전히 선택적인 컴포넌트라는 점도 주목할 만합니다. 오프스크린 렌더링만 필요한 경우에는 필요하지 않습니다. Vulkan에서는 OpenGL에서 필요했던 것처럼 보이지 않는 윈도우를 만드는 등의 해킹 없이도 이것이 가능합니다.

## 윈도우 서피스 생성

디버그 콜백 바로 아래에 `surface` 클래스 멤버를 추가하는 것으로 시작합니다.

```c++
VkSurfaceKHR surface;
```

`VkSurfaceKHR` 객체와 그 사용은 플랫폼에 독립적이지만, 생성은 그렇지 않습니다. 윈도우 시스템 세부 사항에 의존하기 때문입니다. 예를 들어, Windows에서는 `HWND`와 `HMODULE` 핸들이 필요합니다. 따라서 플랫폼별 extension이 있으며, Windows에서는 `VK_KHR_win32_surface`라고 하며 이 역시 `glfwGetRequiredInstanceExtensions`의 목록에 자동으로 포함됩니다.

Windows에서 서피스를 생성하는 데 이 플랫폼별 extension을 어떻게 사용하는지 보여드리겠지만, 이 튜토리얼에서는 실제로 사용하지는 않을 것입니다. GLFW 같은 라이브러리를 사용하면서 다시 플랫폼별 코드를 사용하는 것은 의미가 없기 때문입니다. GLFW는 실제로 플랫폼 차이를 처리해주는 `glfwCreateWindowSurface`를 제공합니다. 그래도 이를 사용하기 전에 내부에서 어떤 일이 일어나는지 보는 것이 좋습니다.

네이티브 플랫폼 함수에 접근하려면 상단의 include를 다음과 같이 업데이트해야 합니다:

```c++
#define VK_USE_PLATFORM_WIN32_KHR
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>
#define GLFW_EXPOSE_NATIVE_WIN32
#include <GLFW/glfw3native.h>
```

윈도우 서피스는 Vulkan 객체이므로, 채워야 할 `VkWin32SurfaceCreateInfoKHR` 구조체가 함께 제공됩니다. 이는 `hwnd`와 `hinstance` 두 가지 중요한 매개변수를 가집니다. 이들은 윈도우와 프로세스의 핸들입니다.

```c++
VkWin32SurfaceCreateInfoKHR createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_WIN32_SURFACE_CREATE_INFO_KHR;
createInfo.hwnd = glfwGetWin32Window(window);
createInfo.hinstance = GetModuleHandle(nullptr);
```

`glfwGetWin32Window` 함수는 GLFW 윈도우 객체에서 원시 `HWND`를 가져오는 데 사용됩니다. `GetModuleHandle` 호출은 현재 프로세스의 `HINSTANCE` 핸들을 반환합니다.

그 후 `vkCreateWin32SurfaceKHR`로 서피스를 생성할 수 있으며, 여기에는 인스턴스, 서피스 생성 세부 정보, 커스텀 할당자, 그리고 서피스 핸들을 저장할 변수에 대한 매개변수가 포함됩니다. 기술적으로 이는 WSI extension 함수이지만 매우 일반적으로 사용되어 표준 Vulkan 로더에 포함되어 있으므로, 다른 extension과 달리 명시적으로 로드할 필요가 없습니다.

```c++
if (vkCreateWin32SurfaceKHR(instance, &createInfo, nullptr, &surface) != VK_SUCCESS) {
    throw std::runtime_error("failed to create window surface!");
}
```

이 과정은 Linux와 같은 다른 플랫폼에서도 비슷합니다. X11에서는 `vkCreateXcbSurfaceKHR`가 XCB 연결과 윈도우를 생성 세부 정보로 받습니다.

`glfwCreateWindowSurface` 함수는 각 플랫폼마다 다른 구현으로 정확히 이 작업을 수행합니다. 이제 이를 우리 프로그램에 통합해보겠습니다. 인스턴스 생성과 `setupDebugMessenger` 직후 `initVulkan`에서 호출될 `createSurface` 함수를 추가합니다.

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
}

void createSurface() {

}
```

GLFW 호출은 구조체 대신 간단한 매개변수를 받으므로 함수의 구현이 매우 간단합니다:

```c++
void createSurface() {
    if (glfwCreateWindowSurface(instance, window, nullptr, &surface) != VK_SUCCESS) {
        throw std::runtime_error("failed to create window surface!");
    }
}
```

매개변수는 `VkInstance`, GLFW 윈도우 포인터, 커스텀 할당자, 그리고 `VkSurfaceKHR` 변수에 대한 포인터입니다. 단순히 관련 플랫폼 호출의 `VkResult`를 전달합니다. GLFW는 서피스를 파괴하기 위한 특별한 함수를 제공하지 않지만, 원래 API를 통해 쉽게 할 수 있습니다:

```c++
void cleanup() {
    ...
    vkDestroySurfaceKHR(instance, surface, nullptr);
    vkDestroyInstance(instance, nullptr);
    ...
}
```

서피스가 인스턴스보다 먼저 파괴되도록 해야 합니다.

## 프레젠테이션 지원 쿼리하기

Vulkan 구현이 윈도우 시스템 통합을 지원하더라도, 시스템의 모든 장치가 이를 지원하는 것은 아닙니다. 따라서 `isDeviceSuitable`을 확장하여 장치가 우리가 생성한 서피스에 이미지를 표시할 수 있는지 확인해야 합니다. 프레젠테이션은 큐별 기능이므로, 실제로는 우리가 생성한 서피스에 대한 프레젠테이션을 지원하는 큐 패밀리를 찾는 문제입니다.

드로잉 명령을 지원하는 큐 패밀리와 프레젠테이션을 지원하는 큐 패밀리가 겹치지 않을 수 있습니다. 따라서 `QueueFamilyIndices` 구조체를 수정하여 별도의 프레젠테이션 큐가 있을 수 있음을 고려해야 합니다:

```c++
struct QueueFamilyIndices {
    std::optional<uint32_t> graphicsFamily;
    std::optional<uint32_t> presentFamily;

    bool isComplete() {
        return graphicsFamily.has_value() && presentFamily.has_value();
    }
};
```

다음으로, `findQueueFamilies` 함수를 수정하여 우리의 윈도우 서피스에 프레젠테이션할 수 있는 큐 패밀리를 찾아보겠습니다. 이를 확인하는 함수는 `vkGetPhysicalDeviceSurfaceSupportKHR`이며, 물리 장치, 큐 패밀리 인덱스, 서피스를 매개변수로 받습니다. `VK_QUEUE_GRAPHICS_BIT`와 같은 루프에 이 호출을 추가합니다:

```c++
VkBool32 presentSupport = false;
vkGetPhysicalDeviceSurfaceSupportKHR(device, i, surface, &presentSupport);
```

그런 다음 boolean 값을 확인하고 프레젠테이션 패밀리 큐 인덱스를 저장합니다:

```c++
if (presentSupport) {
    indices.presentFamily = i;
}
```

결국 이들이 같은 큐 패밀리가 될 가능성이 매우 높지만, 프로그램 전체에서 일관된 접근을 위해 별도의 큐인 것처럼 다룰 것입니다. 그럼에도 성능 향상을 위해 드로잉과 프레젠테이션을 같은 큐에서 지원하는 물리 장치를 명시적으로 선호하는 로직을 추가할 수 있습니다.

## 프레젠테이션 큐 생성하기

남은 것은 논리 장치 생성 절차를 수정하여 프레젠테이션 큐를 생성하고 `VkQueue` 핸들을 검색하는 것입니다. 핸들을 위한 멤버 변수를 추가합니다:

```c++
VkQueue presentQueue;
```

다음으로, 두 패밀리 모두에서 큐를 생성하기 위해 여러 `VkDeviceQueueCreateInfo` 구조체가 필요합니다. 이를 위한 우아한 방법은 필요한 큐를 위한 모든 고유한 큐 패밀리의 집합을 만드는 것입니다:

```c++
#include <set>

...

QueueFamilyIndices indices = findQueueFamilies(physicalDevice);

std::vector<VkDeviceQueueCreateInfo> queueCreateInfos;
std::set<uint32_t> uniqueQueueFamilies = {indices.graphicsFamily.value(), indices.presentFamily.value()};

float queuePriority = 1.0f;
for (uint32_t queueFamily : uniqueQueueFamilies) {
    VkDeviceQueueCreateInfo queueCreateInfo{};
    queueCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
    queueCreateInfo.queueFamilyIndex = queueFamily;
    queueCreateInfo.queueCount = 1;
    queueCreateInfo.pQueuePriorities = &queuePriority;
    queueCreateInfos.push_back(queueCreateInfo);
}
```

그리고 `VkDeviceCreateInfo`를 수정하여 벡터를 가리키도록 합니다:

```c++
createInfo.queueCreateInfoCount = static_cast<uint32_t>(queueCreateInfos.size());
createInfo.pQueueCreateInfos = queueCreateInfos.data();
```

큐 패밀리가 같다면 인덱스를 한 번만 전달하면 됩니다. 마지막으로, 큐 핸들을 검색하는 호출을 추가합니다:

```c++
vkGetDeviceQueue(device, indices.presentFamily.value(), 0, &presentQueue);
```

큐 패밀리가 같다면 두 핸들은 이제 같은 값을 가질 가능성이 높습니다. 다음 장에서는 스왑 체인과 이것이 어떻게 서피스에 이미지를 표시할 수 있게 해주는지 살펴보겠습니다.

[C++ 코드](/code/05_window_surface.cpp)