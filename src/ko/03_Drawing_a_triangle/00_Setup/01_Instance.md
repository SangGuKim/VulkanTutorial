# 인스턴스

## 인스턴스 생성

가장 먼저 해야 할 일은 *인스턴스*를 생성하여 Vulkan 라이브러리를 초기화하는 것입니다. 인스턴스는 애플리케이션과 Vulkan 라이브러리 사이의 연결고리이며, 인스턴스를 생성할 때 드라이버에게 애플리케이션에 대한 몇 가지 세부 정보를 지정하게 됩니다.

먼저 `createInstance` 함수를 추가하고, 이를 `initVulkan` 함수 내에서 호출합니다.

```c++
void initVulkan() {
    createInstance();
}
```

추가로 인스턴스 핸들을 저장할 데이터 멤버를 추가합니다:

```c++
private:
VkInstance instance;
```

이제 인스턴스를 생성하기 위해, 애플리케이션에 관한 정보를 담은 구조체를 먼저 채워야 합니다. 이 데이터는 기술적으로 선택 사항이지만, 드라이버에게 특정 애플리케이션에 최적화할 수 있는 유용한 정보를 제공할 수 있습니다 (예: 특정 특수 동작을 가진 잘 알려진 그래픽 엔진을 사용하기 때문에). 이 구조체는 `VkApplicationInfo`라고 불립니다:

```c++
void createInstance() {
    VkApplicationInfo appInfo{};
    appInfo.sType = VK_STRUCTURE_TYPE_APPLICATION_INFO;
    appInfo.pApplicationName = "Hello Triangle";
    appInfo.applicationVersion = VK_MAKE_VERSION(1, 0, 0);
    appInfo.pEngineName = "No Engine";
    appInfo.engineVersion = VK_MAKE_VERSION(1, 0, 0);
    appInfo.apiVersion = VK_API_VERSION_1_0;
}
```

앞서 언급했듯이, Vulkan의 많은 구조체는 `sType` 멤버에 타입을 명시적으로 지정할 것을 요구합니다. 또한 이는 앞으로 확장 정보(extension information)를 가리킬 수 있는 `pNext` 멤버가 있는 여러 구조체 중 하나입니다. 여기서는 `nullptr`로 두기 위해 값 초기화를 사용하고 있습니다.

Vulkan에서는 많은 정보가 함수 매개변수 대신 구조체를 통해 전달되며, 인스턴스를 생성하는 데 충분한 정보를 제공하기 위해 한 개의 구조체를 더 채워야 합니다. 이 다음 구조체는 선택 사항이 아니며, Vulkan 드라이버에게 우리가 사용하고자 하는 글로벌 확장과 검증 레이어를 알려줍니다. 여기서 "글로벌"이라는 것은 이들이 특정 장치가 아니라 전체 프로그램에 적용된다는 의미이며, 이는 다음 몇 장에서 명확해질 것입니다.

```c++
VkInstanceCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;
createInfo.pApplicationInfo = &appInfo;
```

처음 두 매개변수는 이해하기 쉽습니다. 다음 두 항목은 원하는 글로벌 확장을 지정합니다. 개요 장에서 언급했듯이, Vulkan은 플랫폼에 구애받지 않는 API이므로 창 시스템과의 인터페이스를 위해 확장이 필요합니다. GLFW에는 이러한 작업에 필요한 확장을 반환하는 편리한 내장 함수가 있으며, 이를 구조체에 전달할 수 있습니다:

```c++
uint32_t glfwExtensionCount = 0;
const char** glfwExtensions;

glfwExtensions = glfwGetRequiredInstanceExtensions(&glfwExtensionCount);

createInfo.enabledExtensionCount = glfwExtensionCount;
createInfo.ppEnabledExtensionNames = glfwExtensions;
```

구조체의 마지막 두 멤버는 활성화할 글로벌 검증 레이어를 결정합니다. 이들에 대해서는 다음 장에서 더 자세히 다룰 예정이므로, 지금은 이들을 비워두면 됩니다.

```c++
createInfo.enabledLayerCount = 0;
```

이제 인스턴스를 생성하기 위해 Vulkan이 필요로 하는 모든 사항을 지정하였으며, 마침내 `vkCreateInstance` 호출을 할 수 있습니다:

```c++
VkResult result = vkCreateInstance(&createInfo, nullptr, &instance);
```

보시다시피, Vulkan에서 객체 생성 함수의 매개변수가 따르는 일반적인 패턴은 다음과 같습니다:

* 생성 정보가 담긴 구조체에 대한 포인터  
* 사용자 정의 할당자 콜백에 대한 포인터 (이 튜토리얼에서는 항상 `nullptr`)  
* 새 객체의 핸들을 저장할 변수를 가리키는 포인터

모든 과정이 순조롭게 진행되었다면, 인스턴스의 핸들이 `VkInstance` 클래스 멤버에 저장됩니다. 거의 모든 Vulkan 함수는 `VK_SUCCESS` 또는 에러 코드를 반환하는 `VkResult` 타입의 값을 반환합니다. 인스턴스가 성공적으로 생성되었는지 확인하기 위해 결과 값을 저장할 필요 없이, 성공 값에 대한 검사를 사용하면 됩니다:

```c++
if (vkCreateInstance(&createInfo, nullptr, &instance) != VK_SUCCESS) {
    throw std::runtime_error("failed to create instance!");
}
```

이제 프로그램을 실행하여 인스턴스가 성공적으로 생성되었는지 확인하세요.

---

## VK_ERROR_INCOMPATIBLE_DRIVER 에러 발생

최신 MoltenVK SDK를 사용하는 MacOS 환경에서는 `vkCreateInstance` 호출에서 `VK_ERROR_INCOMPATIBLE_DRIVER` 에러가 반환될 수 있습니다. [Getting Start Notes](https://vulkan.lunarg.com/doc/sdk/1.3.216.0/mac/getting_started.html)에 따르면, 1.3.216 버전의 Vulkan SDK부터는 `VK_KHR_PORTABILITY_subset` 확장이 필수입니다.

이 에러를 해결하기 위해, 먼저 `VkInstanceCreateInfo` 구조체의 flags에 `VK_INSTANCE_CREATE_ENUMERATE_PORTABILITY_BIT_KHR` 비트를 추가한 후, 인스턴스 활성화 확장 목록에 `VK_KHR_PORTABILITY_ENUMERATION_EXTENSION_NAME`을 추가하세요.

일반적으로 코드는 다음과 같이 될 수 있습니다:

```c++
...

std::vector<const char*> requiredExtensions;

for(uint32_t i = 0; i < glfwExtensionCount; i++) {
    requiredExtensions.emplace_back(glfwExtensions[i]);
}

requiredExtensions.emplace_back(VK_KHR_PORTABILITY_ENUMERATION_EXTENSION_NAME);

createInfo.flags |= VK_INSTANCE_CREATE_ENUMERATE_PORTABILITY_BIT_KHR;

createInfo.enabledExtensionCount = (uint32_t) requiredExtensions.size();
createInfo.ppEnabledExtensionNames = requiredExtensions.data();

if (vkCreateInstance(&createInfo, nullptr, &instance) != VK_SUCCESS) {
    throw std::runtime_error("failed to create instance!");
}
```

---

## 확장 지원 확인하기

`vkCreateInstance` 문서를 살펴보면, 가능한 에러 코드 중 하나가 `VK_ERROR_EXTENSION_NOT_PRESENT`임을 알 수 있습니다. 우리는 단순히 요구하는 확장을 지정하고, 해당 에러 코드가 반환되면 종료할 수 있습니다. 이는 창 시스템 인터페이스와 같은 필수 확장에는 타당하지만, 선택적 기능을 확인하고자 할 경우에는 어떻게 해야 할까요?

인스턴스를 생성하기 전에 지원되는 확장 목록을 가져오기 위해 `vkEnumerateInstanceExtensionProperties` 함수가 있습니다. 이 함수는 확장의 개수를 저장할 변수를 가리키는 포인터와, 확장의 세부 정보를 저장할 `VkExtensionProperties` 배열을 인자로 받습니다. 또한, 특정 검증 레이어로 확장을 필터링할 수 있는 선택적 첫 번째 매개변수를 받는데, 이는 지금은 무시하겠습니다.

확장 세부 정보를 저장할 배열을 할당하기 위해 먼저 확장의 개수를 알아야 합니다. 후자의 매개변수를 비워두면 확장의 개수만 요청할 수 있습니다:

```c++
uint32_t extensionCount = 0;
vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);
```

이제 확장 세부 정보를 저장할 배열을 할당합니다 (`<vector>` 포함):

```c++
std::vector<VkExtensionProperties> extensions(extensionCount);
```

마지막으로 확장 세부 정보를 쿼리합니다:

```c++
vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, extensions.data());
```

각 `VkExtensionProperties` 구조체에는 확장의 이름과 버전이 포함되어 있습니다. 간단한 for 루프를 사용하여 이들을 나열할 수 있습니다 (`\t`는 들여쓰기를 위한 탭입니다):

```c++
std::cout << "available extensions:\n";

for (const auto& extension : extensions) {
    std::cout << '\t' << extension.extensionName << '\n';
}
```

Vulkan 지원에 대한 세부 정보를 제공하고자 한다면, 이 코드를 `createInstance` 함수에 추가할 수 있습니다. 도전 과제로, `glfwGetRequiredInstanceExtensions`가 반환한 모든 확장이 지원되는 확장 목록에 포함되어 있는지 확인하는 함수를 만들어 보세요.

---

## 정리하기

`VkInstance`는 프로그램 종료 직전에만 파괴되어야 합니다. `cleanup` 함수 내에서 `vkDestroyInstance` 함수를 사용하여 파괴할 수 있습니다:

```c++
void cleanup() {
    vkDestroyInstance(instance, nullptr);

    glfwDestroyWindow(window);

    glfwTerminate();
}
```

`vkDestroyInstance` 함수의 매개변수들은 이해하기 쉽습니다. 앞 장에서 언급했듯이, Vulkan의 할당 및 해제 함수들은 선택적인 할당자 콜백을 가지는데, 이는 `nullptr`를 전달하여 무시합니다. 이후 장들에서 생성할 다른 모든 Vulkan 리소스들은 인스턴스가 파괴되기 전에 정리되어야 합니다.

인스턴스 생성 이후의 더 복잡한 단계로 진행하기 전에, [검증 레이어](!en/Drawing_a_triangle/Setup/Validation_layers)를 확인하여 디버깅 옵션을 평가할 시간입니다.

[C++ 코드](/code/01_instance_creation.cpp)