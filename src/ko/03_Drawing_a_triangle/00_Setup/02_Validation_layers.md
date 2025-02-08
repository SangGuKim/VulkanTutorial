# 검증 레이어

## 검증 레이어란 무엇인가?

Vulkan API는 최소한의 드라이버 오버헤드를 목표로 설계되었으며, 이 목표의 한 표현으로 기본적으로 매우 제한된 오류 검사가 API 내에 포함되어 있습니다. 열거형(enum)을 잘못된 값으로 설정하거나 필수 매개변수에 null 포인터를 전달하는 것과 같은 간단한 실수들도 명시적으로 처리되지 않으며, 단순히 크래시나 정의되지 않은 동작(undefined behavior)을 초래할 뿐입니다. Vulkan은 사용자가 수행하는 모든 작업에 대해 매우 명시적으로 작성하도록 요구하기 때문에, 새로운 GPU 기능을 사용하면서 논리 디바이스 생성 시 이를 요청하는 것을 잊는 등 작은 실수를 범하기 쉽습니다.

그러나 이것이 API에 이러한 검사를 추가할 수 없다는 의미는 아닙니다. Vulkan은 *검증 레이어(validation layers)* 라고 알려진 우아한 시스템을 도입하여 이러한 기능을 제공합니다. 검증 레이어는 선택적으로 사용할 수 있는 구성 요소로, Vulkan 함수 호출에 후킹되어 추가 작업을 수행합니다. 검증 레이어에서 수행하는 일반적인 작업은 다음과 같습니다:

* 매개변수의 값을 명세(specification)와 비교하여 잘못 사용되었는지 확인
* 객체의 생성과 소멸을 추적하여 리소스 누수를 감지
* 호출이 발생한 스레드를 추적하여 스레드 안전성을 확인
* 모든 호출 및 해당 매개변수를 표준 출력에 로깅
* 프로파일링 및 재생을 위해 Vulkan 호출을 추적

다음은 진단용 검증 레이어에서의 함수 구현 예시입니다:

```c++
VkResult vkCreateInstance(
    const VkInstanceCreateInfo* pCreateInfo,
    const VkAllocationCallbacks* pAllocator,
    VkInstance* instance) {

    if (pCreateInfo == nullptr || instance == nullptr) {
        log("필수 매개변수에 null 포인터가 전달되었습니다!");
        return VK_ERROR_INITIALIZATION_FAILED;
    }

    return real_vkCreateInstance(pCreateInfo, pAllocator, instance);
}
```

이러한 검증 레이어는 원하는 디버깅 기능을 모두 포함하도록 자유롭게 쌓을 수 있습니다. 예를 들어, 디버그 빌드에서는 검증 레이어를 활성화하고 릴리즈 빌드에서는 완전히 비활성화함으로써 양쪽의 장점을 모두 누릴 수 있습니다!

Vulkan은 기본적으로 어떠한 검증 레이어도 내장하고 있지 않지만, LunarG Vulkan SDK는 일반적인 오류를 검사하는 훌륭한 레이어 세트를 제공합니다. 이 레이어들은 완전히 [오픈 소스](https://github.com/KhronosGroup/Vulkan-ValidationLayers)이며, 어떤 종류의 실수를 검사하는지 확인하거나 기여할 수도 있습니다. 검증 레이어를 사용하는 것은 정의되지 않은 동작에 우연히 의존하여 애플리케이션이 다양한 드라이버에서 깨지는 것을 방지하는 가장 좋은 방법입니다.

검증 레이어는 시스템에 설치되어 있을 때만 사용할 수 있습니다. 예를 들어, LunarG 검증 레이어는 Vulkan SDK가 설치된 PC에서만 사용 가능합니다.

이전에는 Vulkan에서 인스턴스 전용과 디바이스 전용, 두 종류의 검증 레이어가 존재했습니다. 인스턴스 레이어는 인스턴스와 같은 전역 Vulkan 객체와 관련된 호출만 검사하고, 디바이스 전용 레이어는 특정 GPU와 관련된 호출만 검사하는 것이 목적이었습니다. 하지만 디바이스 전용 레이어는 이제 더 이상 사용되지 않으며, 인스턴스 검증 레이어가 모든 Vulkan 호출에 적용됩니다. 사양 문서에서는 호환성을 위해 디바이스 레벨에서도 검증 레이어를 활성화할 것을 권장하는데, 일부 구현에서는 이를 요구합니다. 우리는 논리 디바이스 레벨에서도 인스턴스와 동일한 레이어를 지정할 것이며, 이는 [나중에 살펴볼 논리 디바이스와 큐](!en/Drawing_a_triangle/Setup/Logical_device_and_queues)에서 확인할 수 있습니다.


## 검증 레이어 사용하기

이번 섹션에서는 Vulkan SDK에서 제공하는 표준 진단 레이어를 활성화하는 방법을 살펴보겠습니다. 확장(extension)과 마찬가지로, 검증 레이어도 이름을 지정하여 활성화해야 합니다. 유용한 표준 검증 기능은 SDK에 포함된 `VK_LAYER_KHRONOS_validation` 레이어에 모두 번들되어 있습니다.

먼저, 활성화할 레이어와 활성화 여부를 지정하기 위해 두 개의 구성 변수를 프로그램에 추가합니다. 저는 프로그램이 디버그 모드로 컴파일되는지 여부에 따라 해당 값을 설정하도록 선택했습니다. `NDEBUG` 매크로는 C++ 표준의 일부로 "디버그가 아님"을 의미합니다.

```c++
const uint32_t WIDTH = 800;
const uint32_t HEIGHT = 600;

const std::vector<const char*> validationLayers = {
    "VK_LAYER_KHRONOS_validation"
};

#ifdef NDEBUG
    const bool enableValidationLayers = false;
#else
    const bool enableValidationLayers = true;
#endif
```

그 다음, 요청한 모든 레이어가 사용 가능한지 확인하는 `checkValidationLayerSupport` 함수를 추가합니다. 먼저 `vkEnumerateInstanceLayerProperties` 함수를 사용하여 사용 가능한 모든 레이어를 나열합니다. 이 함수의 사용법은 인스턴스 생성 챕터에서 다룬 `vkEnumerateInstanceExtensionProperties`와 동일합니다.

```c++
bool checkValidationLayerSupport() {
    uint32_t layerCount;
    vkEnumerateInstanceLayerProperties(&layerCount, nullptr);

    std::vector<VkLayerProperties> availableLayers(layerCount);
    vkEnumerateInstanceLayerProperties(&layerCount, availableLayers.data());

    return false;
}
```

다음으로, `validationLayers`에 있는 모든 레이어가 `availableLayers` 목록에 존재하는지 확인합니다. 이때 `strcmp`를 사용하기 위해 `<cstring>`을 포함해야 할 수도 있습니다.

```c++
for (const char* layerName : validationLayers) {
    bool layerFound = false;

    for (const auto& layerProperties : availableLayers) {
        if (strcmp(layerName, layerProperties.layerName) == 0) {
            layerFound = true;
            break;
        }
    }

    if (!layerFound) {
        return false;
    }
}

return true;
```

이제 이 함수를 `createInstance` 함수 내에서 사용할 수 있습니다:

```c++
void createInstance() {
    if (enableValidationLayers && !checkValidationLayerSupport()) {
        throw std::runtime_error("검증 레이어가 요청되었으나 사용 가능하지 않습니다!");
    }

    ...
}
```

디버그 모드에서 프로그램을 실행하여 오류가 발생하지 않는지 확인하세요. 만약 오류가 발생한다면 FAQ를 확인해 보시기 바랍니다.

마지막으로, `VkInstanceCreateInfo` 구조체 인스턴스 생성 시 검증 레이어 이름들을 포함하도록 수정합니다:

```c++
if (enableValidationLayers) {
    createInfo.enabledLayerCount = static_cast<uint32_t>(validationLayers.size());
    createInfo.ppEnabledLayerNames = validationLayers.data();
} else {
    createInfo.enabledLayerCount = 0;
}
```

체크가 성공하면 `vkCreateInstance`는 `VK_ERROR_LAYER_NOT_PRESENT` 오류를 반환하지 않아야 하지만, 실제로 프로그램을 실행하여 확인하는 것이 좋습니다.


## 메시지 콜백

검증 레이어는 기본적으로 디버그 메시지를 표준 출력으로 출력하지만, 프로그램 내에서 명시적인 콜백을 제공하여 직접 처리할 수도 있습니다. 이를 통해 모든 메시지가 반드시 (치명적인) 오류가 아니라는 점에서 원하는 종류의 메시지를 선택적으로 확인할 수 있습니다. 만약 지금 당장 이 작업을 수행하고 싶지 않다면 이 장의 마지막 섹션으로 건너뛰어도 됩니다.

메시지와 관련된 세부 정보를 처리하기 위한 콜백을 설정하려면, `VK_EXT_debug_utils` 확장을 사용하여 콜백이 포함된 디버그 메신저를 설정해야 합니다.

먼저, 검증 레이어가 활성화되어 있는지 여부에 따라 필요한 확장 목록을 반환하는 `getRequiredExtensions` 함수를 생성합니다:

```c++
std::vector<const char*> getRequiredExtensions() {
    uint32_t glfwExtensionCount = 0;
    const char** glfwExtensions;
    glfwExtensions = glfwGetRequiredInstanceExtensions(&glfwExtensionCount);

    std::vector<const char*> extensions(glfwExtensions, glfwExtensions + glfwExtensionCount);

    if (enableValidationLayers) {
        extensions.push_back(VK_EXT_DEBUG_UTILS_EXTENSION_NAME);
    }

    return extensions;
}
```

GLFW에서 지정한 확장은 항상 필요하지만, 디버그 메신저 확장은 조건에 따라 추가됩니다. 여기서 사용한 `VK_EXT_DEBUG_UTILS_EXTENSION_NAME` 매크로는 리터럴 문자열 `"VK_EXT_debug_utils"`와 동일하며, 이 매크로를 사용하면 오타를 방지할 수 있습니다.

이제 이 함수를 `createInstance`에서 사용할 수 있습니다:

```c++
auto extensions = getRequiredExtensions();
createInfo.enabledExtensionCount = static_cast<uint32_t>(extensions.size());
createInfo.ppEnabledExtensionNames = extensions.data();
```

프로그램을 실행하여 `VK_ERROR_EXTENSION_NOT_PRESENT` 오류가 발생하지 않는지 확인하세요. 이 확장이 존재하는지 별도로 확인할 필요는 없으며, 이는 검증 레이어의 사용 가능성에 의해 암시되기 때문입니다.

이제 디버그 콜백 함수가 어떻게 생겼는지 살펴보겠습니다. `PFN_vkDebugUtilsMessengerCallbackEXT` 프로토타입을 가진 정적 멤버 함수 `debugCallback`을 추가하세요. `VKAPI_ATTR`와 `VKAPI_CALL`은 Vulkan이 올바른 서명으로 이 함수를 호출할 수 있도록 보장합니다.

```c++
static VKAPI_ATTR VkBool32 VKAPI_CALL debugCallback(
    VkDebugUtilsMessageSeverityFlagBitsEXT messageSeverity,
    VkDebugUtilsMessageTypeFlagsEXT messageType,
    const VkDebugUtilsMessengerCallbackDataEXT* pCallbackData,
    void* pUserData) {

    std::cerr << "검증 레이어: " << pCallbackData->pMessage << std::endl;

    return VK_FALSE;
}
```

첫 번째 매개변수는 메시지의 심각도를 지정하며, 다음 플래그들 중 하나의 값을 가질 수 있습니다:

* `VK_DEBUG_UTILS_MESSAGE_SEVERITY_VERBOSE_BIT_EXT`: 진단 메시지  
* `VK_DEBUG_UTILS_MESSAGE_SEVERITY_INFO_BIT_EXT`: 리소스 생성 등의 정보 메시지  
* `VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT`: 오류는 아닐 수 있으나 애플리케이션의 버그일 가능성이 높은 동작에 대한 메시지  
* `VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT`: 잘못된 동작에 대한 메시지로, 크래시를 유발할 수 있음

이 열거형의 값들은 메시지의 심각도가 특정 수준 이상인지를 비교 연산으로 확인할 수 있도록 구성되어 있습니다. 예를 들어:

```c++
if (messageSeverity >= VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT) {
    // 메시지가 충분히 중요하여 표시됨
}
```

`messageType` 매개변수는 다음 값을 가질 수 있습니다:

* `VK_DEBUG_UTILS_MESSAGE_TYPE_GENERAL_BIT_EXT`: 사양이나 성능과 관련 없는 이벤트 발생  
* `VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT`: 사양 위반 또는 잠재적 실수를 나타내는 이벤트 발생  
* `VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT`: Vulkan의 비최적 사용 가능성

`pCallbackData` 매개변수는 메시지의 세부 정보를 담고 있는 `VkDebugUtilsMessengerCallbackDataEXT` 구조체를 가리키며, 가장 중요한 멤버는 다음과 같습니다:

* `pMessage`: null 종료 문자열 형식의 디버그 메시지  
* `pObjects`: 메시지와 관련된 Vulkan 객체 핸들 배열  
* `objectCount`: 배열 내 객체의 개수

마지막으로, `pUserData` 매개변수는 콜백 설정 시 지정했던 포인터를 포함하며, 이를 통해 사용자가 원하는 데이터를 전달할 수 있습니다.

콜백 함수는 해당 Vulkan 호출을 중단할지 여부를 나타내는 부울 값을 반환합니다. 만약 콜백이 true를 반환하면, 해당 호출은 `VK_ERROR_VALIDATION_FAILED_EXT` 오류와 함께 중단됩니다. 이는 일반적으로 검증 레이어 자체를 테스트할 때만 사용되므로, 항상 `VK_FALSE`를 반환해야 합니다.

남은 작업은 Vulkan에 이 콜백 함수에 대해 알리는 것입니다. 다소 놀랍게도, Vulkan의 디버그 콜백도 명시적으로 생성 및 소멸해야 하는 핸들로 관리됩니다. 이러한 콜백은 *디버그 메신저*의 일부이며, 원하는 만큼 여러 개를 생성할 수 있습니다. `instance` 바로 아래에 이 핸들을 위한 클래스 멤버를 추가합니다:

```c++
VkDebugUtilsMessengerEXT debugMessenger;
```

이제 `createInstance` 호출 직후, `initVulkan`에서 호출될 `setupDebugMessenger` 함수를 추가합니다:

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
}

void setupDebugMessenger() {
    if (!enableValidationLayers) return;

}
```

메신저와 그 콜백에 대한 세부 정보를 담을 구조체를 채워야 합니다:

```c++
VkDebugUtilsMessengerCreateInfoEXT createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_DEBUG_UTILS_MESSENGER_CREATE_INFO_EXT;
createInfo.messageSeverity = VK_DEBUG_UTILS_MESSAGE_SEVERITY_VERBOSE_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT;
createInfo.messageType = VK_DEBUG_UTILS_MESSAGE_TYPE_GENERAL_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT;
createInfo.pfnUserCallback = debugCallback;
createInfo.pUserData = nullptr; // 선택 사항
```

`messageSeverity` 필드는 콜백이 호출되길 원하는 모든 심각도 유형을 지정할 수 있습니다. 여기서는 `VK_DEBUG_UTILS_MESSAGE_SEVERITY_INFO_BIT_EXT`를 제외한 모든 유형을 지정하여, 자세한 일반 디버그 정보는 제외하고 가능한 문제에 대한 알림만 받도록 했습니다.

유사하게 `messageType` 필드는 콜백이 알림을 받을 메시지 유형을 필터링할 수 있게 해줍니다. 여기서는 모든 유형을 단순히 활성화했습니다. 필요에 따라 일부를 비활성화할 수 있습니다.

마지막으로, `pfnUserCallback` 필드는 콜백 함수의 포인터를 지정합니다. 선택적으로 `pUserData` 필드에 포인터를 전달할 수 있으며, 이 포인터는 콜백 함수의 `pUserData` 매개변수를 통해 전달됩니다. 예를 들어, `HelloTriangleApplication` 클래스의 포인터를 전달할 수도 있습니다.

검증 레이어 메시지 및 디버그 콜백을 구성하는 방법에는 이보다 훨씬 다양한 옵션이 있지만, 이 설정은 튜토리얼을 시작하기 위한 좋은 기본 설정입니다. 가능한 설정에 대한 자세한 정보는 [확장 사양](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap50.html#VK_EXT_debug_utils)을 참조하세요.

이 구조체는 `vkCreateDebugUtilsMessengerEXT` 함수에 전달되어 `VkDebugUtilsMessengerEXT` 객체를 생성하는 데 사용됩니다. 안타깝게도 이 함수는 확장 함수이므로 자동으로 로드되지 않습니다. `vkGetInstanceProcAddr`를 사용하여 직접 주소를 찾아야 합니다. 이를 위해 백그라운드에서 처리할 프록시 함수를 생성할 것입니다. 이 함수는 `HelloTriangleApplication` 클래스 정의 바로 위에 추가했습니다.

```c++
VkResult CreateDebugUtilsMessengerEXT(VkInstance instance, const VkDebugUtilsMessengerCreateInfoEXT* pCreateInfo, const VkAllocationCallbacks* pAllocator, VkDebugUtilsMessengerEXT* pDebugMessenger) {
    auto func = (PFN_vkCreateDebugUtilsMessengerEXT) vkGetInstanceProcAddr(instance, "vkCreateDebugUtilsMessengerEXT");
    if (func != nullptr) {
        return func(instance, pCreateInfo, pAllocator, pDebugMessenger);
    } else {
        return VK_ERROR_EXTENSION_NOT_PRESENT;
    }
}
```

`vkGetInstanceProcAddr` 함수는 해당 함수가 로드되지 않으면 `nullptr`을 반환합니다. 이제 이 함수를 호출하여, 해당 확장 객체를 생성할 수 있습니다:

```c++
if (CreateDebugUtilsMessengerEXT(instance, &createInfo, nullptr, &debugMessenger) != VK_SUCCESS) {
    throw std::runtime_error("디버그 메신저 설정에 실패했습니다!");
}
```

두 번째 마지막 매개변수는 앞서 `nullptr`로 설정한 선택적 할당자 콜백입니다. 그 외의 매개변수는 상당히 직관적입니다. 디버그 메신저는 우리 Vulkan 인스턴스와 그 레이어에 특화되어 있으므로 첫 번째 인수로 명시적으로 지정해야 합니다. 이후 다른 *자식* 객체에서도 이 패턴을 보게 될 것입니다.

`VkDebugUtilsMessengerEXT` 객체도 `vkDestroyDebugUtilsMessengerEXT` 호출을 통해 정리되어야 합니다. 마찬가지로 `vkCreateDebugUtilsMessengerEXT`처럼 이 함수도 명시적으로 로드되어야 합니다.

`CreateDebugUtilsMessengerEXT` 바로 아래에 또 다른 프록시 함수를 생성합니다:

```c++
void DestroyDebugUtilsMessengerEXT(VkInstance instance, VkDebugUtilsMessengerEXT debugMessenger, const VkAllocationCallbacks* pAllocator) {
    auto func = (PFN_vkDestroyDebugUtilsMessengerEXT) vkGetInstanceProcAddr(instance, "vkDestroyDebugUtilsMessengerEXT");
    if (func != nullptr) {
        func(instance, debugMessenger, pAllocator);
    }
}
```

이 함수가 정적 클래스 함수이거나 클래스 외부의 함수임을 확인하세요. 그런 다음 `cleanup` 함수 내에서 이를 호출합니다:

```c++
void cleanup() {
    if (enableValidationLayers) {
        DestroyDebugUtilsMessengerEXT(instance, debugMessenger, nullptr);
    }

    vkDestroyInstance(instance, nullptr);

    glfwDestroyWindow(window);

    glfwTerminate();
}
```

## 인스턴스 생성 및 소멸 디버깅

검증 레이어를 통한 디버깅을 프로그램에 추가했지만, 아직 모든 것을 다루지는 않았습니다. `vkCreateDebugUtilsMessengerEXT` 호출은 유효한 인스턴스가 생성된 후에 이루어져야 하며, `vkDestroyDebugUtilsMessengerEXT`는 인스턴스가 소멸되기 전에 호출되어야 합니다. 이 때문에 현재로서는 `vkCreateInstance`와 `vkDestroyInstance` 호출 중 발생하는 문제를 디버깅할 수 없습니다.

그러나 [확장 문서](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/appendices/VK_EXT_debug_utils.adoc#examples)를 주의 깊게 읽어보면, 이 두 함수 호출에 대해 별도의 디버그 유틸 메신저를 생성하는 방법이 있음을 알 수 있습니다. 이는 `VkInstanceCreateInfo`의 `pNext` 확장 필드에 `VkDebugUtilsMessengerCreateInfoEXT` 구조체의 포인터를 전달하기만 하면 됩니다. 먼저 메신저 생성 정보를 별도의 함수로 추출합니다:

```c++
void populateDebugMessengerCreateInfo(VkDebugUtilsMessengerCreateInfoEXT& createInfo) {
    createInfo = {};
    createInfo.sType = VK_STRUCTURE_TYPE_DEBUG_UTILS_MESSENGER_CREATE_INFO_EXT;
    createInfo.messageSeverity = VK_DEBUG_UTILS_MESSAGE_SEVERITY_VERBOSE_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT;
    createInfo.messageType = VK_DEBUG_UTILS_MESSAGE_TYPE_GENERAL_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT;
    createInfo.pfnUserCallback = debugCallback;
}
```

...

```c++
void setupDebugMessenger() {
    if (!enableValidationLayers) return;

    VkDebugUtilsMessengerCreateInfoEXT createInfo;
    populateDebugMessengerCreateInfo(createInfo);

    if (CreateDebugUtilsMessengerEXT(instance, &createInfo, nullptr, &debugMessenger) != VK_SUCCESS) {
        throw std::runtime_error("디버그 메신저 설정에 실패했습니다!");
    }
}
```

이제 이 함수를 `createInstance` 함수 내에서도 재사용할 수 있습니다:

```c++
void createInstance() {
    ...

    VkInstanceCreateInfo createInfo{};
    createInfo.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;
    createInfo.pApplicationInfo = &appInfo;

    ...

    VkDebugUtilsMessengerCreateInfoEXT debugCreateInfo{};
    if (enableValidationLayers) {
        createInfo.enabledLayerCount = static_cast<uint32_t>(validationLayers.size());
        createInfo.ppEnabledLayerNames = validationLayers.data();

        populateDebugMessengerCreateInfo(debugCreateInfo);
        createInfo.pNext = (VkDebugUtilsMessengerCreateInfoEXT*) &debugCreateInfo;
    } else {
        createInfo.enabledLayerCount = 0;

        createInfo.pNext = nullptr;
    }

    if (vkCreateInstance(&createInfo, nullptr, &instance) != VK_SUCCESS) {
        throw std::runtime_error("인스턴스 생성에 실패했습니다!");
    }
}
```

여기서 `debugCreateInfo` 변수는 `vkCreateInstance` 호출 전에 소멸되지 않도록 if 문 밖에 배치되었습니다. 이렇게 추가 디버그 메신저를 생성하면, `vkCreateInstance`와 `vkDestroyInstance` 호출 시 자동으로 사용되며 이후 정리됩니다.


## 테스트

이제 검증 레이어의 동작을 확인하기 위해 의도적으로 실수를 만들어 보겠습니다. `cleanup` 함수에서 `DestroyDebugUtilsMessengerEXT` 호출을 임시로 제거한 후 프로그램을 실행하세요. 프로그램이 종료되면 다음과 유사한 메시지가 출력될 것입니다:

![](/images/validation_layer_test.png)

>만약 메시지가 보이지 않는다면 [설치를 확인](https://vulkan.lunarg.com/doc/view/1.2.131.1/windows/getting_started.html#user-content-verify-the-installation)해 보세요.

어떤 호출이 메시지를 트리거했는지 확인하고 싶다면, 메시지 콜백에 중단점을 추가한 후 스택 트레이스를 확인할 수 있습니다.

## 설정

검증 레이어의 동작을 제어하기 위한 설정은 `VkDebugUtilsMessengerCreateInfoEXT` 구조체에 지정된 플래그 외에도 훨씬 더 많습니다. Vulkan SDK의 `Config` 디렉토리를 확인해 보세요. 그곳에서 레이어 설정 방법을 설명하는 `vk_layer_settings.txt` 파일을 찾을 수 있습니다.

자신의 애플리케이션에 대해 레이어 설정을 구성하려면, 해당 파일을 프로젝트의 `Debug` 및 `Release` 디렉토리로 복사한 후 원하는 동작을 설정하기 위한 지침을 따르십시오. 하지만 이 튜토리얼의 나머지 부분에서는 기본 설정을 사용한다고 가정하겠습니다.

이 튜토리얼 전반에 걸쳐 저는 검증 레이어가 얼마나 유용한지, 그리고 Vulkan을 사용할 때 자신이 무엇을 하고 있는지 정확히 아는 것이 얼마나 중요한지를 보여주기 위해 의도적인 실수를 몇 가지 포함시킬 것입니다. 이제 [시스템 내의 Vulkan 디바이스](!en/Drawing_a_triangle/Setup/Physical_devices_and_queue_families)를 살펴볼 시간입니다.

[C++ 코드](/code/02_validation_layers.cpp)