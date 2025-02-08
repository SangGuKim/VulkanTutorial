# 자주 묻는 질문 (FAQ)

이 페이지에서는 Vulkan 애플리케이션 개발 중에 마주칠 수 있는 일반적인 문제들에 대한 해결책을 제시합니다.

## 코어 검증 레이어에서 액세스 위반 오류가 발생합니다

MSI 애프터버너 / 리바튜너 통계 서버가 실행 중인지 확인하세요. 이 프로그램들은 Vulkan과의 호환성 문제가 있습니다.

## 검증 레이어에서 메시지가 보이지 않습니다 / 검증 레이어를 사용할 수 없습니다

먼저 프로그램이 종료된 후 터미널이 열려 있는지 확인하여 검증 레이어가 오류를 출력할 기회가 있는지 확인하세요. Visual Studio에서는 프로그램을 F5가 아닌 Ctrl-F5로 실행하고, Linux에서는 터미널 창에서 프로그램을 실행합니다. 메시지가 여전히 없고 검증 레이어가 활성화되어 있는지 확신하는 경우, "설치 확인" 지침을 따라 Vulkan SDK가 올바르게 설치되어 있는지 확인하세요. 또한 `VK_LAYER_KHRONOS_validation` 레이어를 지원하려면 SDK 버전이 최소 1.1.106.0 이상인지 확인하세요.

## vkCreateSwapchainKHR가 SteamOverlayVulkanLayer64.dll에서 오류를 발생시킵니다

이는 Steam 클라이언트 베타의 호환성 문제로 보입니다. 몇 가지 가능한 해결책이 있습니다:
- Steam 베타 프로그램에서 탈퇴합니다.
- `DISABLE_VK_LAYER_VALVE_steam_overlay_1` 환경 변수를 `1`로 설정합니다.
- `HKEY_LOCAL_MACHINE\SOFTWARE\Khronos\Vulkan\ImplicitLayers` 하위에 있는 Steam 오버레이 Vulkan 레이어 항목을 삭제합니다.

예시:

![](/images/steam_layers_env.png)

## vkCreateInstance가 VK_ERROR_INCOMPATIBLE_DRIVER로 실패합니다

MacOS를 사용 중이고 최신 MoltenVK SDK를 사용하는 경우 `vkCreateInstance`가 `VK_ERROR_INCOMPATIBLE_DRIVER` 오류를 반환할 수 있습니다. 이는 [Vulkan SDK 버전 1.3.216 이상](https://vulkan.lunarg.com/doc/sdk/1.3.216.0/mac/getting_started.html)에서 MoltenVK를 사용하기 위해 `VK_KHR_PORTABILITY_subset` 확장을 활성화해야 하기 때문입니다. 현재 MoltenVK는 완전히 호환되지 않습니다.

`VkInstanceCreateInfo`에 `VK_INSTANCE_CREATE_ENUMERATE_PORTABILITY_BIT_KHR` 플래그를 추가하고 인스턴스 확장 목록에 `VK_KHR_PORTABILITY_ENUMERATION_EXTENSION_NAME`을 추가해야 합니다.

코드 예시:

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