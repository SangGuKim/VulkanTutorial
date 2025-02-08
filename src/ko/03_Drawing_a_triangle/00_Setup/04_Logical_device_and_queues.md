# 논리적 장치와 큐

## 소개

물리 장치를 선택한 후에는 이와 인터페이스하기 위한 논리적 장치를 설정해야 합니다. 논리적 장치 생성 과정은 인스턴스 생성 과정과 비슷하며 우리가 사용하고자 하는 기능들을 명시합니다. 또한 사용 가능한 큐 패밀리들을 조회했으니 이제 어떤 큐를 생성할지 지정해야 합니다. 요구사항이 다양한 경우 동일한 물리 장치에서 여러 논리적 장치를 생성할 수도 있습니다.

먼저 논리적 장치 핸들을 저장할 새로운 클래스 멤버를 추가합니다.

```c++
VkDevice device;
```

다음으로, `initVulkan`에서 호출될 `createLogicalDevice` 함수를 추가합니다.

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    pickPhysicalDevice();
    createLogicalDevice();
}

void createLogicalDevice() {

}
```

## 생성할 큐 지정하기

논리적 장치를 생성하는 것은 다시 한번 구조체에 여러 세부 사항을 지정하는 것을 포함하며, 그 중 첫 번째는 `VkDeviceQueueCreateInfo`입니다. 이 구조체는 단일 큐 패밀리에 대해 우리가 원하는 큐의 개수를 설명합니다. 현재는 그래픽스 기능이 있는 큐에만 관심이 있습니다.

```c++
QueueFamilyIndices indices = findQueueFamilies(physicalDevice);

VkDeviceQueueCreateInfo queueCreateInfo{};
queueCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
queueCreateInfo.queueFamilyIndex = indices.graphicsFamily.value();
queueCreateInfo.queueCount = 1;
```

현재 사용 가능한 드라이버들은 각 큐 패밀리에 대해 소수의 큐만 생성하도록 허용하며, 실제로 하나 이상은 필요하지 않습니다. 그 이유는 여러 스레드에서 모든 커맨드 버퍼를 생성한 다음 메인 스레드에서 단일 저오버헤드 호출로 한 번에 모두 제출할 수 있기 때문입니다.

Vulkan은 `0.0`과 `1.0` 사이의 부동소수점 숫자를 사용하여 큐에 우선순위를 할당하여 커맨드 버퍼 실행 스케줄링에 영향을 줄 수 있게 합니다. 큐가 하나뿐이더라도 이는 필수입니다:

```c++
float queuePriority = 1.0f;
queueCreateInfo.pQueuePriorities = &queuePriority;
```

## 사용할 device 기능 지정하기

다음으로 지정할 정보는 우리가 사용할 device 기능들의 집합입니다. 이는 이전 챕터에서 `vkGetPhysicalDeviceFeatures`로 지원 여부를 조회했던 geometry shader와 같은 기능들입니다. 현재는 특별한 것이 필요하지 않으므로 단순히 정의하고 모든 것을 `VK_FALSE`로 두면 됩니다. Vulkan으로 더 흥미로운 작업을 하기 시작할 때 이 구조체로 돌아오겠습니다.

```c++
VkPhysicalDeviceFeatures deviceFeatures{};
```

## 논리적 장치 생성하기

이전 두 구조체가 준비되었으니, 이제 메인 `VkDeviceCreateInfo` 구조체를 채우기 시작할 수 있습니다.

```c++
VkDeviceCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO;
```

먼저 큐 생성 정보와 device 기능 구조체에 대한 포인터를 추가합니다:

```c++
createInfo.pQueueCreateInfos = &queueCreateInfo;
createInfo.queueCreateInfoCount = 1;

createInfo.pEnabledFeatures = &deviceFeatures;
```

나머지 정보는 `VkInstanceCreateInfo` 구조체와 유사하며 extension과 검증 레이어를 지정해야 합니다. 차이점은 이번에는 이것들이 device 특정적이라는 것입니다.

device 특정 extension의 예시로는 `VK_KHR_swapchain`이 있으며, 이는 해당 device에서 렌더링된 이미지를 창에 표시할 수 있게 해줍니다. 시스템에는 이러한 기능이 없는 Vulkan device가 있을 수 있습니다. 예를 들어 컴퓨트 연산만 지원하는 경우가 있을 수 있습니다. 이 extension에 대해서는 스왑 체인 챕터에서 다시 다루겠습니다.

이전 Vulkan 구현에서는 인스턴스와 device 특정 검증 레이어를 구분했지만, [이제는 더 이상 그렇지 않습니다](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap40.html#extendingvulkan-layers-devicelayerdeprecation). 이는 최신 구현에서는 `VkDeviceCreateInfo`의 `enabledLayerCount`와 `ppEnabledLayerNames` 필드가 무시된다는 의미입니다. 하지만 이전 구현과의 호환성을 위해 여전히 이를 설정하는 것이 좋습니다:

```c++
createInfo.enabledExtensionCount = 0;

if (enableValidationLayers) {
    createInfo.enabledLayerCount = static_cast<uint32_t>(validationLayers.size());
    createInfo.ppEnabledLayerNames = validationLayers.data();
} else {
    createInfo.enabledLayerCount = 0;
}
```

현재는 device 특정 extension이 필요하지 않습니다.

이제 적절하게 이름 지어진 `vkCreateDevice` 함수를 호출하여 논리적 장치를 인스턴스화할 준비가 되었습니다.

```c++
if (vkCreateDevice(physicalDevice, &createInfo, nullptr, &device) != VK_SUCCESS) {
    throw std::runtime_error("failed to create logical device!");
}
```

매개변수는 인터페이스할 물리 장치, 방금 지정한 큐와 사용 정보, 선택적 할당 콜백 포인터, 그리고 논리적 장치 핸들을 저장할 변수에 대한 포인터입니다. 인스턴스 생성 함수와 마찬가지로 이 호출도 존재하지 않는 extension을 활성화하거나 지원되지 않는 기능의 사용을 지정하는 경우 오류를 반환할 수 있습니다.

장치는 `cleanup`에서 `vkDestroyDevice` 함수로 파괴되어야 합니다:

```c++
void cleanup() {
    vkDestroyDevice(device, nullptr);
    ...
}
```

논리적 장치는 인스턴스와 직접 상호작용하지 않기 때문에 매개변수로 포함되지 않습니다.

## 큐 핸들 검색하기

큐는 논리적 장치와 함께 자동으로 생성되지만, 아직 이와 인터페이스할 핸들이 없습니다. 먼저 그래픽스 큐에 대한 핸들을 저장할 클래스 멤버를 추가합니다:

```c++
VkQueue graphicsQueue;
```

Device 큐는 device가 파괴될 때 암시적으로 정리되므로 `cleanup`에서 별도로 처리할 필요가 없습니다.

각 큐 패밀리에 대한 큐 핸들을 검색하기 위해 `vkGetDeviceQueue` 함수를 사용할 수 있습니다. 매개변수는 논리적 장치, 큐 패밀리, 큐 인덱스, 그리고 큐 핸들을 저장할 변수에 대한 포인터입니다. 이 패밀리에서 하나의 큐만 생성하므로 인덱스 `0`을 사용하면 됩니다.

```c++
vkGetDeviceQueue(device, indices.graphicsFamily.value(), 0, &graphicsQueue);
```

논리적 장치와 큐 핸들이 있으니 이제 실제로 그래픽 카드를 사용하여 작업을 수행할 수 있습니다! 다음 몇 챕터에서는 결과를 창 시스템에 표시하기 위한 리소스를 설정하겠습니다.

[C++ 코드](/code/04_logical_device.cpp)