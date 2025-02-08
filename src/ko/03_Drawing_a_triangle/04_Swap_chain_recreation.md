# 스왑 체인(Swap Chain) 재생성

## 소개

우리가 지금 가지고 있는 애플리케이션은 삼각형을 성공적으로 그립니다. 하지만 아직 제대로 처리하지 못하는 몇 가지 상황이 있습니다. 창(surface)이 변경되어 스왑 체인이 더 이상 호환되지 않을 수 있습니다. 이러한 상황을 일으킬 수 있는 이유 중 하나는 창 크기의 변경입니다. 이러한 이벤트를 캐치하고 스왑 체인을 재생성해야 합니다.

## 스왑 체인 재생성

스왑 체인이나 창 크기에 따라 달라지는 객체들을 위한 생성 함수를 호출하는 새로운 `recreateSwapChain` 함수를 만듭니다.

```c++
void recreateSwapChain() {
    vkDeviceWaitIdle(device);

    createSwapChain();
    createImageViews();
    createFramebuffers();
}
```

우리는 먼저 `vkDeviceWaitIdle`을 호출합니다. 마지막 장에서처럼, 여전히 사용 중일 수 있는 리소스를 만지지 않아야 하기 때문입니다. 당연히 스왑 체인 자체를 재생성해야 합니다. 이미지 뷰는 스왑 체인 이미지에 직접 기반하기 때문에 재생성해야 합니다. 마지막으로, 프레임버퍼는 스왑 체인 이미지에 직접 의존하기 때문에 재생성되어야 합니다.

이 객체들의 이전 버전을 재생성하기 전에 정리하는 것이 좋으므로, 일부 정리 코드를 `recreateSwapChain` 함수에서 호출할 수 있는 별도의 함수로 이동해야 합니다. `cleanupSwapChain`이라고 부릅시다:

```c++
void cleanupSwapChain() {

}

void recreateSwapChain() {
    vkDeviceWaitIdle(device);

    cleanupSwapChain();

    createSwapChain();
    createImageViews();
    createFramebuffers();
}
```

여기서는 단순화를 위해 렌더 패스(render pass)를 재생성하지 않습니다. 이론적으로 애플리케이션의 수명 동안 스왑 체인 이미지 형식이 변경될 수 있습니다(예: 표준 범위 모니터에서 HDR(high dynamic range) 모니터로 창을 이동할 때). 이는 애플리케이션에게 동적 범위 사이의 변경을 제대로 반영하도록 렌더 패스를 재생성해야 할 수도 있습니다.

스왑 체인 새로 고침의 일부로 재생성되는 모든 객체의 정리 코드를 `cleanup`에서 `cleanupSwapChain`으로 이동하겠습니다:

```c++
void cleanupSwapChain() {
    for (auto framebuffer : swapChainFramebuffers) {
        vkDestroyFramebuffer(device, framebuffer, nullptr);
    }

    for (auto imageView : swapChainImageViews) {
        vkDestroyImageView(device, imageView, nullptr);
    }

    vkDestroySwapchainKHR(device, swapChain, nullptr);
}

void cleanup() {
    cleanupSwapChain();

    vkDestroyPipeline(device, graphicsPipeline, nullptr);
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);

    vkDestroyRenderPass(device, renderPass, nullptr);

    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        vkDestroySemaphore(device, renderFinishedSemaphores[i], nullptr);
        vkDestroySemaphore(device, imageAvailableSemaphores[i], nullptr);
        vkDestroyFence(device, inFlightFences[i], nullptr);
    }

    vkDestroyCommandPool(device, commandPool, nullptr);

    vkDestroyDevice(device, nullptr);

    if (enableValidationLayers) {
        DestroyDebugUtilsMessengerEXT(instance, debugMessenger, nullptr);
    }

    vkDestroySurfaceKHR(instance, surface, nullptr);
    vkDestroyInstance(instance, nullptr);

    glfwDestroyWindow(window);

    glfwTerminate();
}
```

`chooseSwapExtent`에서 이미 새 창 해상도를 조회하여 스왑 체인 이미지가 새로운(올바른) 크기를 갖도록 했기 때문에 `chooseSwapExtent`를 수정할 필요가 없습니다(스왑 체인을 생성할 때 픽셀 단위로 표면의 해상도를 얻기 위해 `glfwGetFramebufferSize`를 이미 사용했음을 기억하세요).

스왑 체인을 재생성하는 것은 이것뿐입니다! 그러나 이 접근 방식의 단점은 새 스왑 체인을 생성하기 전에 모든 렌더링을 중지해야 한다는 것입니다. 오래된 스왑 체인의 이미지에서 그리기 명령이 여전히 진행 중인 동안 새 스왑 체인을 생성할 수 있습니다. `VkSwapchainCreateInfoKHR` 구조체의 `oldSwapChain` 필드에 이전 스왑 체인을 전달하고 그것을 사용을 마친 후에 오래된 스왑 체인을 파괴해야 합니다.

## 최적이 아니거나 오래된 스왑 체인

이제 스왑 체인 재생성이 필요한 시기를 파악하고 새로운 `recreateSwapChain` 함수를 호출해야 합니다. 다행히 Vulkan은 일반적으로 프레젠테이션 도중 스왑 체인이 더 이상 적합하지 않다고 알려줍니다. `vkAcquireNextImageKHR` 및 `vkQueuePresentKHR` 함수는 다음과 같은 특별한 값을 반환하여 이를 나타낼 수 있습니다.

* `VK_ERROR_OUT_OF_DATE_KHR`: 스왑 체인이 표면과 호환되지 않아 더 이상 렌더링에 사용할 수 없습니다. 일반적으로 창 크기가 변경된 후에 발생합니다.
* `VK_SUBOPTIMAL_KHR`: 스왑 체인은 여전히 표면에 성공적으로 표시할 수 있지만 표면 속성이 정확하게 일치하지는 않습니다.

```c++
VkResult result = vkAcquireNextImageKHR(device, swapChain, UINT64_MAX, imageAvailableSemaphores[currentFrame], VK_NULL_HANDLE, &imageIndex);

if (result == VK_ERROR_OUT_OF_DATE_KHR) {
    recreateSwapChain();
    return;
} else if (result != VK_SUCCESS && result != VK_SUBOPTIMAL_KHR) {
    throw std::runtime_error("failed to acquire swap chain image!");
}
```

이미지를 획득하려고 할 때 스왑 체인이 오래되었다면 더 이상 그것에 표시할 수 없습니다. 따라서 즉시 스왑 체인을 재생성하고 다음 `drawFrame` 호출에서 다시 시도해야 합니다.

스왑 체인이 최적이 아닌 경우에도 그렇게 할 수 있지만, 이미 이미지를 획득했기 때문에 그 경우에는 계속 진행하기로 결정했습니다. `VK_SUCCESS` 및 `VK_SUBOPTIMAL_KHR` 모두 "성공" 반환 코드로 간주됩니다.

```c++
result = vkQueuePresentKHR(presentQueue, &presentInfo);

if (result == VK_ERROR_OUT_OF_DATE_KHR || result == VK_SUBOPTIMAL_KHR) {
    recreateSwapChain();
} else if (result != VK_SUCCESS) {
    throw std::runtime_error("failed to present swap chain image!");
}

currentFrame = (currentFrame + 1) % MAX_FRAMES_IN_FLIGHT;
```

`vkQueuePresentKHR` 함수는 동일한 값과 동일한 의미를 반환합니다. 이 경우에도 스왑 체인이 최적이 아니면

 재생성합니다. 왜냐하면 가능한 최상의 결과를 원하기 때문입니다.

## 데드락 수정

이제 코드를 실행하려고 하면 데드락에 빠질 수 있습니다. 코드를 디버깅하면 애플리케이션이 `vkWaitForFences`에 도달했지만 그 이상으로 계속되지 않는 것을 발견할 수 있습니다. 이는 `vkAcquireNextImageKHR`가 `VK_ERROR_OUT_OF_DATE_KHR`를 반환하면 스왑체인을 재생성한 후 `drawFrame`에서 반환하기 때문입니다. 그러나 그 전에 현재 프레임의 펜스가 대기되었고 재설정되었습니다. 즉시 반환하면 실행할 작업이 제출되지 않고 펜스가 결코 신호되지 않아 `vkWaitForFences`가 영원히 중단됩니다.

다행히 간단한 해결책이 있습니다. 확실히 작업을 제출할 것이라는 것을 알기 전까지 펜스를 재설정하지 않도록 합니다. 따라서 일찍 반환하면 펜스는 여전히 신호되어 있고, 다음에 동일한 펜스 객체를 사용할 때 `vkWaitForFences`가 데드락에 빠지지 않습니다.

`drawFrame`의 시작 부분은 이제 다음과 같아야 합니다:
```c++
vkWaitForFences(device, 1, &inFlightFences[currentFrame], VK_TRUE, UINT64_MAX);

uint32_t imageIndex;
VkResult result = vkAcquireNextImageKHR(device, swapChain, UINT64_MAX, imageAvailableSemaphores[currentFrame], VK_NULL_HANDLE, &imageIndex);

if (result == VK_ERROR_OUT_OF_DATE_KHR) {
    recreateSwapChain();
    return;
} else if (result != VK_SUCCESS && result != VK_SUBOPTIMAL_KHR) {
    throw std::runtime_error("failed to acquire swap chain image!");
}

// 작업을 제출할 때만 펜스를 재설정합니다.
vkResetFences(device, 1, &inFlightFences[currentFrame]);
```

## 명시적으로 크기 조정 처리

많은 드라이버와 플랫폼은 창 크기 조정 후 자동으로 `VK_ERROR_OUT_OF_DATE_KHR`를 트리거하지만, 이것이 발생한다는 보장은 없습니다. 그렇기 때문에 크기 조정을 명시적으로 처리하는 추가 코드를 작성할 것입니다. 먼저 크기 조정이 발생했음을 플래그하는 새로운 멤버 변수를 추가하세요:

```c++
std::vector<VkFence> inFlightFences;

bool framebufferResized = false;
```

그런 다음 `drawFrame` 함수를 수정하여 이 플래그도 확인하도록 합니다:

```c++
if (result == VK_ERROR_OUT_OF_DATE_KHR || result == VK_SUBOPTIMAL_KHR || framebufferResized) {
    framebufferResized = false;
    recreateSwapChain();
} else if (result != VK_SUCCESS) {
    ...
}
```

`vkQueuePresentKHR` 이후에 이 작업을 수행하는 것이 중요합니다. 그렇지 않으면 세마포어가 일관된 상태에 있지 않을 수 있으며, 신호된 세마포어가 제대로 대기되지 않을 수 있습니다. 이제 실제로 크기 조정을 감지하려면 GLFW 프레임워크의 `glfwSetFramebufferSizeCallback` 함수를 사용하여 콜백을 설정할 수 있습니다:

```c++
void initWindow() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);

    window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
    glfwSetFramebufferSizeCallback(window, framebufferResizeCallback);
}

static void framebufferResizeCallback(GLFWwindow* window, int width, int height) {

}
```

콜백으로 `static` 함수를 생성하는 이유는 GLFW가 우리 `HelloTriangleApplication` 인스턴스에 올바른 `this` 포인터로 멤버 함수를 제대로 호출하는 방법을 모르기 때문입니다.

그러나 콜백에서 `GLFWwindow`에 대한 참조를 얻을 수 있으며, 임의의 포인터를 그 안에 저장할 수 있는 또 다른 GLFW 함수가 있습니다: `glfwSetWindowUserPointer`:

```c++
window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
glfwSetWindowUserPointer(window, this);
glfwSetFramebufferSizeCallback(window, framebufferResizeCallback);
```

이제 이 값을 콜백 내에서 `glfwGetWindowUserPointer`를 사용하여 검색하여 플래그를 올바르게 설정할 수 있습니다:

```c++
static void framebufferResizeCallback(GLFWwindow* window, int width, int height) {
    auto app = reinterpret_cast<HelloTriangleApplication*>(glfwGetWindowUserPointer(window));
    app->framebufferResized = true;
}
```

이제 프로그램을 실행하고 창 크기를 조절하여 프레임버퍼가 창 크기에 맞게 제대로 조절되는지 확인해 보세요.

## 최소화 처리

스왑 체인이 오래되는 또 다른 경우는 특별한 종류의 창 크기 조절입니다: 창 최소화입니다. 이 경우는 특별합니다. 왜냐하면 `0`의 프레임 버퍼 크기로 결과를 낳기 때문입니다. 이 튜토리얼에서는 창이 다시 전경에 있을 때까지 일시 중지하여 `recreateSwapChain` 함수를 확장함으로써 이를 처리할 것입니다:

```c++
void recreateSwapChain() {
    int width = 0, height = 0;
    glfwGetFramebufferSize(window, &width, &height);
    while (width == 0 || height == 0) {
        glfwGetFramebufferSize(window, &width, &height);
        glfwWaitEvents();
    }

    vkDeviceWaitIdle(device);

    ...
}
```

`glfwGetFramebufferSize`의 초기 호출은 크기가 이미 정확하고 `glfwWaitEvents`가 기다릴 것이 없을 경우를 처리합니다.

축하합니다, 이제 매우 잘 동작하는 첫 Vulkan 프로그램을 완성했습니다! 다음 장에서는 정점 셰이더에서 하드코딩된 정점을 제거하고 실제 정점 버퍼를 사용하도록 변경할 것입니다.

[C++ 코드](/code/17_swap_chain_recreation.cpp) /
[버텍스 셰이더(vertex shader)](/code/09_shader_base.vert) /
[프래그먼트 셰이더(fragment shader)](/code/09_shader_base.frag)