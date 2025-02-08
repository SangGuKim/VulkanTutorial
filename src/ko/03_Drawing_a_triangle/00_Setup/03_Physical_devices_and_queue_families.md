# 물리적 장치와 큐 패밀리

## 물리적 장치 선택하기

VkInstance를 통해 Vulkan 라이브러리를 초기화한 후에는 우리가 필요로 하는 기능들을 지원하는 그래픽 카드를 시스템에서 찾아 선택해야 합니다. 사실 여러 개의 그래픽 카드를 선택해서 동시에 사용할 수도 있지만, 이 튜토리얼에서는 우리가 필요로 하는 기능을 갖춘 첫 번째 그래픽 카드만 사용하도록 하겠습니다.

`pickPhysicalDevice` 함수를 추가하고 `initVulkan` 함수에서 이를 호출하도록 하겠습니다.

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    pickPhysicalDevice();
}

void pickPhysicalDevice() {

}
```

선택하게 될 그래픽 카드는 새로운 클래스 멤버로 추가된 VkPhysicalDevice 핸들에 저장됩니다. 이 객체는 VkInstance가 파괴될 때 암시적으로 파괴되므로, `cleanup` 함수에서 별도로 처리할 필요가 없습니다.

```c++
VkPhysicalDevice physicalDevice = VK_NULL_HANDLE;
```

그래픽 카드를 나열하는 것은 extension을 나열하는 것과 매우 비슷하며, 먼저 개수만 조회하는 것으로 시작합니다.

```c++
uint32_t deviceCount = 0;
vkEnumeratePhysicalDevices(instance, &deviceCount, nullptr);
```

Vulkan을 지원하는 device가 0개라면 더 진행할 이유가 없습니다.

```c++
if (deviceCount == 0) {
    throw std::runtime_error("failed to find GPUs with Vulkan support!");
}
```

그렇지 않다면 이제 모든 VkPhysicalDevice 핸들을 담을 배열을 할당할 수 있습니다.

```c++
std::vector<VkPhysicalDevice> devices(deviceCount);
vkEnumeratePhysicalDevices(instance, &deviceCount, devices.data());
```

이제 각각의 device를 평가하여 우리가 수행하고자 하는 작업에 적합한지 확인해야 합니다. 모든 그래픽 카드가 동등하게 만들어지지는 않았기 때문입니다. 이를 위해 새로운 함수를 도입하겠습니다:

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    return true;
}
```

그리고 물리적 장치들 중 어느 것이 우리가 이 함수에 추가할 요구사항을 충족하는지 확인해보겠습니다.

```c++
for (const auto& device : devices) {
    if (isDeviceSuitable(device)) {
        physicalDevice = device;
        break;
    }
}

if (physicalDevice == VK_NULL_HANDLE) {
    throw std::runtime_error("failed to find a suitable GPU!");
}
```

다음 섹션에서는 `isDeviceSuitable` 함수에서 확인할 첫 번째 요구사항들을 소개합니다. 이후 챕터에서 더 많은 Vulkan 기능을 사용하기 시작하면서 이 함수에 더 많은 검사를 추가할 것입니다.

## 기본 device 적합성 검사

device의 적합성을 평가하기 위해 먼저 몇 가지 세부 정보를 조회해볼 수 있습니다. 이름, 타입, 지원하는 Vulkan 버전과 같은 기본적인 device 속성은 vkGetPhysicalDeviceProperties를 사용하여 조회할 수 있습니다.

```c++
VkPhysicalDeviceProperties deviceProperties;
vkGetPhysicalDeviceProperties(device, &deviceProperties);
```

텍스처 압축, 64비트 float, 멀티 뷰포트 렌더링(VR에 유용한)과 같은 선택적 기능의 지원 여부는 vkGetPhysicalDeviceFeatures를 사용하여 조회할 수 있습니다:

```c++
VkPhysicalDeviceFeatures deviceFeatures;
vkGetPhysicalDeviceFeatures(device, &deviceFeatures);
```

device memory와 queue family에 관한 더 많은 세부 정보를 조회할 수 있는데, 이는 다음 섹션에서 다루도록 하겠습니다.

예를 들어, 우리의 애플리케이션이 geometry shader를 지원하는 전용 그래픽 카드에서만 사용 가능하다고 가정해봅시다. 그러면 `isDeviceSuitable` 함수는 다음과 같이 보일 것입니다:

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    VkPhysicalDeviceProperties deviceProperties;
    VkPhysicalDeviceFeatures deviceFeatures;
    vkGetPhysicalDeviceProperties(device, &deviceProperties);
    vkGetPhysicalDeviceFeatures(device, &deviceFeatures);

    return deviceProperties.deviceType == VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU &&
           deviceFeatures.geometryShader;
}
```

device가 적합한지 아닌지만 확인하고 첫 번째 것을 선택하는 대신, 각 device에 점수를 매기고 가장 높은 점수를 가진 것을 선택할 수도 있습니다. 이렇게 하면 전용 그래픽 카드에 더 높은 점수를 주어 우선순위를 둘 수 있지만, 그것만 사용 가능한 경우에는 통합 GPU로 대체할 수 있습니다. 다음과 같이 구현할 수 있습니다:

```c++
#include <map>

...

void pickPhysicalDevice() {
    ...

    // 자동으로 후보들을 점수 순으로 정렬하기 위해 ordered map 사용
    std::multimap<int, VkPhysicalDevice> candidates;

    for (const auto& device : devices) {
        int score = rateDeviceSuitability(device);
        candidates.insert(std::make_pair(score, device));
    }

    // 최고 점수 후보가 적합한지 확인
    if (candidates.rbegin()->first > 0) {
        physicalDevice = candidates.rbegin()->second;
    } else {
        throw std::runtime_error("failed to find a suitable GPU!");
    }
}

int rateDeviceSuitability(VkPhysicalDevice device) {
    ...

    int score = 0;

    // 전용 GPU는 상당한 성능 이점이 있음
    if (deviceProperties.deviceType == VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU) {
        score += 1000;
    }

    // 텍스처의 최대 가능 크기는 그래픽 품질에 영향을 미침
    score += deviceProperties.limits.maxImageDimension2D;

    // 애플리케이션은 geometry shader 없이는 작동할 수 없음
    if (!deviceFeatures.geometryShader) {
        return 0;
    }

    return score;
}
```

이 튜토리얼에서는 이 모든 것을 구현할 필요는 없지만, device 선택 프로세스를 어떻게 설계할 수 있는지에 대한 아이디어를 제공하기 위한 것입니다. 물론 선택 가능한 device들의 이름을 표시하고 사용자가 선택하도록 할 수도 있습니다.

우선은 Vulkan 지원이 필요한 유일한 요구사항이므로 어떤 GPU든 상관없이 진행하겠습니다:

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    return true;
}
```

다음 섹션에서는 확인해야 할 첫 번째 실제 필수 기능에 대해 논의하겠습니다.

## 큐 패밀리

이전에 간단히 언급했듯이 Vulkan에서는 그리기부터 텍스처 업로드까지 거의 모든 작업이 큐에 명령을 제출해야 합니다. 서로 다른 큐 패밀리에서 비롯된 여러 종류의 큐가 있으며, 각 큐 패밀리는 특정 명령들의 부분집합만을 허용합니다. 예를 들어, 컴퓨트 명령만 처리할 수 있는 큐 패밀리나 메모리 전송 관련 명령만 허용하는 큐 패밀리가 있을 수 있습니다.

우리는 device가 어떤 큐 패밀리들을 지원하는지, 그리고 그 중 어떤 것이 우리가 사용하고자 하는 명령들을 지원하는지 확인해야 합니다. 이를 위해 우리가 필요로 하는 모든 큐 패밀리를 찾는 새로운 함수 `findQueueFamilies`를 추가하겠습니다.

현재는 그래픽스 명령을 지원하는 큐만 찾아볼 것이므로, 함수는 다음과 같이 보일 수 있습니다:

```c++
uint32_t findQueueFamilies(VkPhysicalDevice device) {
    // 그래픽스 큐 패밀리를 찾는 로직
}
```

하지만 다음 챕터 중 하나에서 이미 다른 큐를 찾아볼 예정이므로, 이에 대비해 인덱스들을 구조체로 묶는 것이 좋습니다:

```c++
struct QueueFamilyIndices {
    uint32_t graphicsFamily;
};

QueueFamilyIndices findQueueFamilies(VkPhysicalDevice device) {
    QueueFamilyIndices indices;
    // 구조체를 채우기 위한 큐 패밀리 인덱스를 찾는 로직
    return indices;
}
```

하지만 큐 패밀리를 사용할 수 없다면 어떻게 될까요? `findQueueFamilies`에서 예외를 던질 수도 있지만, 이 함수는 device 적합성에 대한 결정을 내리기에 적절한 위치가 아닙니다. 예를 들어, 전용 전송 큐 패밀리가 있는 device를 선호할 수는 있지만 필수 요구사항은 아닐 수 있습니다. 따라서 특정 큐 패밀리가 발견되었는지를 나타내는 방법이 필요합니다.

`uint32_t`의 어떤 값도 이론적으로는 유효한 큐 패밀리 인덱스가 될 수 있기 때문에(`0`도 포함), 매직 값을 사용해 큐 패밀리가 없음을 나타내는 것은 불가능합니다. 다행히도 C++17에서는 값이 존재하는지 여부를 구분할 수 있는 데이터 구조를 도입했습니다:

```c++
#include <optional>

...

std::optional<uint32_t> graphicsFamily;

std::cout << std::boolalpha << graphicsFamily.has_value() << std::endl; // false

graphicsFamily = 0;

std::cout << std::boolalpha << graphicsFamily.has_value() << std::endl; // true
```

`std::optional`은 무언가를 할당할 때까지 값을 포함하지 않는 래퍼입니다. `has_value()` 멤버 함수를 호출하여 언제든지 값을 포함하고 있는지 여부를 확인할 수 있습니다. 이는 다음과 같이 로직을 변경할 수 있다는 의미입니다:

```c++
#include <optional>

...

struct QueueFamilyIndices {
    std::optional<uint32_t> graphicsFamily;
};

QueueFamilyIndices findQueueFamilies(VkPhysicalDevice device) {
    QueueFamilyIndices indices;
    // 찾을 수 있는 큐 패밀리에 인덱스 할당
    return indices;
}
```

이제 `findQueueFamilies`를 실제로 구현하기 시작할 수 있습니다:

```c++
QueueFamilyIndices findQueueFamilies(VkPhysicalDevice device) {
    QueueFamilyIndices indices;

    ...

    return indices;
}
```

큐 패밀리 목록을 검색하는 과정은 예상대로이며 `vkGetPhysicalDeviceQueueFamilyProperties`를 사용합니다:

```c++
uint32_t queueFamilyCount = 0;
vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, nullptr);

std::vector<VkQueueFamilyProperties> queueFamilies(queueFamilyCount);
vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, queueFamilies.data());
```

VkQueueFamilyProperties 구조체는 지원되는 작업 유형과 해당 패밀리를 기반으로 생성할 수 있는 큐의 수를 포함하여 큐 패밀리에 대한 세부 정보를 포함합니다. 우리는 최소한 `VK_QUEUE_GRAPHICS_BIT`를 지원하는 큐 패밀리 하나를 찾아야 합니다.

```c++
int i = 0;
for (const auto& queueFamily : queueFamilies) {
    if (queueFamily.queueFlags & VK_QUEUE_GRAPHICS_BIT) {
        indices.graphicsFamily = i;
    }

    i++;
}
```

이제 이 멋진 큐 패밀리 검색 함수를 가지고 있으니, 이를 `isDeviceSuitable` 함수에서 검사로 사용하여 device가 우리가 사용하고자 하는 명령들을 처리할 수 있는지 확인할 수 있습니다:

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    QueueFamilyIndices indices = findQueueFamilies(device);

    return indices.graphicsFamily.has_value();
}
```

이를 좀 더 편리하게 만들기 위해, 구조체 자체에 일반적인 검사도 추가하겠습니다:

```c++
struct QueueFamilyIndices {
    std::optional<uint32_t> graphicsFamily;

    bool isComplete() {
        return graphicsFamily.has_value();
    }
};

...

bool isDeviceSuitable(VkPhysicalDevice device) {
    QueueFamilyIndices indices = findQueueFamilies(device);

    return indices.isComplete();
}
```

이제 이를 `findQueueFamilies`에서 조기 종료를 위해서도 사용할 수 있습니다:

```c++
for (const auto& queueFamily : queueFamilies) {
    ...

    if (indices.isComplete()) {
        break;
    }

    i++;
}
```

좋습니다, 적절한 물리적 장치를 찾기 위해 지금은 이 정도면 충분합니다! 다음 단계는 이와 인터페이스하기 위한 [논리 장치를 생성](!en/Drawing_a_triangle/Setup/Logical_device_and_queues)하는 것입니다.

[C++ 코드](/code/03_physical_device_selection.cpp)