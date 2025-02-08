# 고정 함수

이전 그래픽스 API들은 그래픽스 파이프라인의 대부분의 스테이지에 대해 기본 상태를 제공했습니다. Vulkan에서는 대부분의 파이프라인 상태를 명시적으로 지정해야 합니다. 이는 변경할 수 없는 파이프라인 상태 객체에 포함되기 때문입니다. 이 장에서는 이러한 고정 함수 작업을 구성하기 위한 모든 구조체를 채워보겠습니다.

## 동적 상태

파이프라인 상태의 *대부분*은 파이프라인 상태에 포함되어야 하지만, 일부 제한된 상태는 실제로 파이프라인을 재생성하지 않고도 드로우 시점에 변경할 수 *있습니다*. 뷰포트의 크기, 선 굵기, 블렌드 상수 등이 그 예입니다. 동적 상태를 사용하고 이러한 속성들을 제외하고 싶다면, 다음과 같이 `VkPipelineDynamicStateCreateInfo` 구조체를 채워야 합니다:

```c++
std::vector<VkDynamicState> dynamicStates = {
    VK_DYNAMIC_STATE_VIEWPORT,
    VK_DYNAMIC_STATE_SCISSOR
};

VkPipelineDynamicStateCreateInfo dynamicState{};
dynamicState.sType = VK_STRUCTURE_TYPE_PIPELINE_DYNAMIC_STATE_CREATE_INFO;
dynamicState.dynamicStateCount = static_cast<uint32_t>(dynamicStates.size());
dynamicState.pDynamicStates = dynamicStates.data();
```

이렇게 하면 이러한 값들의 구성이 무시되고, 드로잉 시점에 데이터를 지정할 수 있게(그리고 필수적으로 지정해야 하게) 됩니다. 이는 더 유연한 설정을 가능하게 하며, 뷰포트와 시저 상태와 같이 파이프라인 상태에 포함될 경우 더 복잡한 설정이 필요한 것들에 대해 매우 일반적입니다.

## 버텍스 입력

`VkPipelineVertexInputStateCreateInfo` 구조체는 버텍스 셰이더에 전달될 버텍스 데이터의 형식을 설명합니다. 이는 크게 두 가지 방식으로 설명됩니다:

* 바인딩: 데이터 간의 간격과 데이터가 버텍스별인지 인스턴스별인지 여부([인스턴싱](https://en.wikipedia.org/wiki/Geometry_instancing) 참조)
* 어트리뷰트 설명: 버텍스 셰이더에 전달되는 어트리뷰트의 타입, 어떤 바인딩에서 로드할지, 어떤 오프셋에서 로드할지

우리는 버텍스 데이터를 버텍스 셰이더에 직접 하드코딩할 것이므로, 현재로서는 로드할 버텍스 데이터가 없음을 지정하기 위해 이 구조체를 다음과 같이 채우겠습니다. 버텍스 버퍼 장에서 이 부분으로 다시 돌아올 것입니다.

```c++
VkPipelineVertexInputStateCreateInfo vertexInputInfo{};
vertexInputInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO;
vertexInputInfo.vertexBindingDescriptionCount = 0;
vertexInputInfo.pVertexBindingDescriptions = nullptr; // 선택적
vertexInputInfo.vertexAttributeDescriptionCount = 0;
vertexInputInfo.pVertexAttributeDescriptions = nullptr; // 선택적
```

`pVertexBindingDescriptions`와 `pVertexAttributeDescriptions` 멤버는 버텍스 데이터를 로드하기 위한 앞서 언급한 세부사항을 설명하는 구조체 배열을 가리킵니다. `createGraphicsPipeline` 함수에서 `shaderStages` 배열 바로 다음에 이 구조체를 추가하세요.

## 입력 어셈블리

`VkPipelineInputAssemblyStateCreateInfo` 구조체는 두 가지를 설명합니다: 버텍스로부터 어떤 종류의 도형이 그려질 것인지와 프리미티브 재시작이 활성화되어야 하는지 여부입니다. 전자는 `topology` 멤버에서 지정되며 다음과 같은 값들을 가질 수 있습니다:

* `VK_PRIMITIVE_TOPOLOGY_POINT_LIST`: 버텍스로부터의 점
* `VK_PRIMITIVE_TOPOLOGY_LINE_LIST`: 재사용 없이 매 2개의 버텍스로부터의 선
* `VK_PRIMITIVE_TOPOLOGY_LINE_STRIP`: 모든 선의 끝 버텍스가 다음 선의 시작 버텍스로 사용됨
* `VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST`: 재사용 없이 매 3개의 버텍스로부터의 삼각형
* `VK_PRIMITIVE_TOPOLOGY_TRIANGLE_STRIP`: 모든 삼각형의 두 번째와 세 번째 버텍스가 다음 삼각형의 첫 두 버텍스로 사용됨

일반적으로 버텍스는 순차적인 순서로 버텍스 버퍼에서 인덱스로 로드되지만, *엘리먼트 버퍼*를 사용하면 사용할 인덱스를 직접 지정할 수 있습니다. 이를 통해 버텍스를 재사용하는 등의 최적화를 수행할 수 있습니다. `primitiveRestartEnable` 멤버를 `VK_TRUE`로 설정하면, `_STRIP` 토폴로지 모드에서 특별한 인덱스인 `0xFFFF` 또는 `0xFFFFFFFF`를 사용하여 선과 삼각형을 분리할 수 있습니다.

이 튜토리얼에서는 삼각형을 그릴 것이므로, 구조체에 다음과 같은 데이터를 사용하겠습니다:

```c++
VkPipelineInputAssemblyStateCreateInfo inputAssembly{};
inputAssembly.sType = VK_STRUCTURE_TYPE_PIPELINE_INPUT_ASSEMBLY_STATE_CREATE_INFO;
inputAssembly.topology = VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST;
inputAssembly.primitiveRestartEnable = VK_FALSE;
```

## 뷰포트와 시저

뷰포트는 기본적으로 출력이 렌더링될 프레임버퍼의 영역을 설명합니다. 이는 거의 항상 `(0, 0)`에서 `(width, height)`까지가 될 것이며, 이 튜토리얼에서도 마찬가지입니다.

```c++
VkViewport viewport{};
viewport.x = 0.0f;
viewport.y = 0.0f;
viewport.width = (float) swapChainExtent.width;
viewport.height = (float) swapChainExtent.height;
viewport.minDepth = 0.0f;
viewport.maxDepth = 1.0f;
```

스왑 체인과 그 이미지들의 크기가 윈도우의 `WIDTH`와 `HEIGHT`와 다를 수 있다는 것을 기억하세요. 스왑 체인 이미지들은 나중에 프레임버퍼로 사용될 것이므로, 우리는 그들의 크기를 사용해야 합니다.

`minDepth`와 `maxDepth` 값은 프레임버퍼에 사용할 깊이 값의 범위를 지정합니다. 이 값들은 `[0.0f, 1.0f]` 범위 내에 있어야 하지만, `minDepth`가 `maxDepth`보다 클 수 있습니다. 특별한 작업을 하지 않는다면, 표준 값인 `0.0f`와 `1.0f`를 사용하면 됩니다.

뷰포트가 이미지에서 프레임버퍼로의 변환을 정의하는 반면, 시저 사각형은 픽셀이 실제로 저장될 영역을 정의합니다. 시저 사각형 외부의 모든 픽셀은 래스터라이저에 의해 폐기됩니다. 이들은 변환이 아닌 필터처럼 작동합니다. 이 차이는 아래 그림에서 설명됩니다. 왼쪽 시저 사각형은 뷰포트보다 크기만 하다면 그 이미지를 만들어낼 수 있는 많은 가능성 중 하나일 뿐임을 주목하세요.

![](/images/viewports_scissors.png)

따라서 전체 프레임버퍼에 그리고 싶다면, 다음과 같이 전체를 덮는 시저 사각형을 지정하면 됩니다:

```c++
VkRect2D scissor{};
scissor.offset = {0, 0};
scissor.extent = swapChainExtent;
```

뷰포트와 시저 사각형은 파이프라인의 정적 부분으로 지정하거나 [동적 상태](#dynamic-state)로 커맨드 버퍼에서 설정할 수 있습니다. 전자가 다른 상태들과 더 일치하지만, 뷰포트와 시저 상태를 동적으로 만드는 것이 더 많은 유연성을 제공하므로 종종 더 편리합니다. 이는 매우 일반적이며 모든 구현체가 성능 저하 없이 이 동적 상태를 처리할 수 있습니다.

동적 뷰포트와 시저 사각형을 선택할 경우 파이프라인에 대해 해당 동적 상태를 활성화해야 합니다:

```c++
std::vector<VkDynamicState> dynamicStates = {
    VK_DYNAMIC_STATE_VIEWPORT,
    VK_DYNAMIC_STATE_SCISSOR
};

VkPipelineDynamicStateCreateInfo dynamicState{};
dynamicState.sType = VK_STRUCTURE_TYPE_PIPELINE_DYNAMIC_STATE_CREATE_INFO;
dynamicState.dynamicStateCount = static_cast<uint32_t>(dynamicStates.size());
dynamicState.pDynamicStates = dynamicStates.data();
```

그리고 파이프라인 생성 시에는 개수만 지정하면 됩니다:

```c++
VkPipelineViewportStateCreateInfo viewportState{};
viewportState.sType = VK_STRUCTURE_TYPE_PIPELINE_VIEWPORT_STATE_CREATE_INFO;
viewportState.viewportCount = 1;
viewportState.scissorCount = 1;
```

실제 뷰포트와 시저 사각형은 나중에 드로잉 시점에 설정됩니다.

동적 상태를 사용하면 단일 커맨드 버퍼 내에서 서로 다른 뷰포트나 시저 사각형을 지정하는 것도 가능합니다.

동적 상태 없이는 뷰포트와 시저 사각형을 `VkPipelineViewportStateCreateInfo` 구조체를 사용하여 파이프라인에서 설정해야 합니다. 이는 이 파이프라인의 뷰포트와 시저 사각형을 변경할 수 없게 만듭니다. 이 값들을 변경하려면 새로운 값으로 새 파이프라인을 생성해야 합니다.

```c++
VkPipelineViewportStateCreateInfo viewportState{};
viewportState.sType = VK_STRUCTURE_TYPE_PIPELINE_VIEWPORT_STATE_CREATE_INFO;
viewportState.viewportCount = 1;
viewportState.pViewports = &viewport;
viewportState.scissorCount = 1;
viewportState.pScissors = &scissor;
```

설정 방법과 관계없이, 일부 그래픽 카드에서는 여러 뷰포트와 시저 사각형을 사용할 수 있습니다. 따라서 구조체 멤버들은 이들의 배열을 참조합니다. 여러 개를 사용하려면 GPU 기능을 활성화해야 합니다(논리적 디바이스 생성 참조).

## 래스터라이저

래스터라이저는 버텍스 셰이더의 버텍스들로 형성된 도형을 가져와서 프래그먼트 셰이더가 색칠할 프래그먼트로 변환합니다. 또한 [깊이 테스트](https://en.wikipedia.org/wiki/Z-buffering), [면 컬링](https://en.wikipedia.org/wiki/Back-face_culling), 시저 테스트를 수행하며, 폴리곤 전체를 채우거나 가장자리만(와이어프레임 렌더링) 프래그먼트를 출력하도록 구성할 수 있습니다. 이 모든 것은 `VkPipelineRasterizationStateCreateInfo` 구조체를 사용하여 구성됩니다.

```c++
VkPipelineRasterizationStateCreateInfo rasterizer{};
rasterizer.sType = VK_STRUCTURE_TYPE_PIPELINE_RASTERIZATION_STATE_CREATE_INFO;
rasterizer.depthClampEnable = VK_FALSE;
```

`depthClampEnable`이 `VK_TRUE`로 설정되면, near 평면과 far 평면을 넘어서는 프래그먼트들이 폐기되는 대신 해당 평면에 고정됩니다. 이는 섀도우 맵과 같은 특수한 경우에 유용합니다. 이를 사용하려면 GPU 기능을 활성화해야 합니다.

```c++
rasterizer.rasterizerDiscardEnable = VK_FALSE;
```

`rasterizerDiscardEnable`이 `VK_TRUE`로 설정되면, 도형이 래스터라이저 스테이지를 전혀 통과하지 않습니다. 이는 기본적으로 프레임버퍼로의 모든 출력을 비활성화합니다.

```c++
rasterizer.polygonMode = VK_POLYGON_MODE_FILL;
```

`polygonMode`는 도형에 대한 프래그먼트 생성 방식을 결정합니다. 다음과 같은 모드들을 사용할 수 있습니다:

* `VK_POLYGON_MODE_FILL`: 프래그먼트로 폴리곤의 영역을 채움
* `VK_POLYGON_MODE_LINE`: 폴리곤의 가장자리를 선으로 그림
* `VK_POLYGON_MODE_POINT`: 폴리곤의 버텍스를 점으로 그림

채우기 모드 이외의 모드를 사용하려면 GPU 기능을 활성화해야 합니다.

```c++
rasterizer.lineWidth = 1.0f;
```

`lineWidth` 멤버는 간단합니다. 프래그먼트 수 단위로 선의 두께를 지정합니다. 지원되는 최대 선 두께는 하드웨어에 따라 다르며, `1.0f`보다 두꺼운 선을 사용하려면 `wideLines` GPU 기능을 활성화해야 합니다.

```c++
rasterizer.cullMode = VK_CULL_MODE_BACK_BIT;
rasterizer.frontFace = VK_FRONT_FACE_CLOCKWISE;
```

`cullMode` 변수는 사용할 면 컬링의 유형을 결정합니다. 컬링을 비활성화하거나, 앞면을 컬링하거나, 뒷면을 컬링하거나, 둘 다 컬링할 수 있습니다. `frontFace` 변수는 앞면으로 간주될 면의 버텍스 순서를 지정하며, 시계 방향이나 반시계 방향이 될 수 있습니다.

```c++
rasterizer.depthBiasEnable = VK_FALSE;
rasterizer.depthBiasConstantFactor = 0.0f; // 선택적
rasterizer.depthBiasClamp = 0.0f; // 선택적
rasterizer.depthBiasSlopeFactor = 0.0f; // 선택적
```

래스터라이저는 상수 값을 추가하거나 프래그먼트의 기울기를 기반으로 바이어스를 주는 방식으로 깊이 값을 변경할 수 있습니다. 이는 때때로 섀도우 매핑에 사용되지만, 우리는 사용하지 않을 것입니다. 그냥 `depthBiasEnable`을 `VK_FALSE`로 설정하세요.

## 멀티샘플링

`VkPipelineMultisampleStateCreateInfo` 구조체는 [안티앨리어싱](https://en.wikipedia.org/wiki/Multisample_anti-aliasing)을 수행하는 방법 중 하나인 멀티샘플링을 구성합니다. 이는 동일한 픽셀에 래스터화되는 여러 폴리곤의 프래그먼트 셰이더 결과를 결합하는 방식으로 작동합니다. 이는 주로 가장자리를 따라 발생하며, 이는 또한 가장 눈에 띄는 앨리어싱 아티팩트가 발생하는 곳이기도 합니다. 하나의 폴리곤만 픽셀에 매핑되는 경우 프래그먼트 셰이더를 여러 번 실행할 필요가 없기 때문에, 단순히 더 높은 해상도로 렌더링한 다음 다운스케일하는 것보다 훨씬 비용이 적게 듭니다. 이를 활성화하려면 GPU 기능을 활성화해야 합니다.

```c++
VkPipelineMultisampleStateCreateInfo multisampling{};
multisampling.sType = VK_STRUCTURE_TYPE_PIPELINE_MULTISAMPLE_STATE_CREATE_INFO;
multisampling.sampleShadingEnable = VK_FALSE;
multisampling.rasterizationSamples = VK_SAMPLE_COUNT_1_BIT;
multisampling.minSampleShading = 1.0f; // 선택적
multisampling.pSampleMask = nullptr; // 선택적
multisampling.alphaToCoverageEnable = VK_FALSE; // 선택적
multisampling.alphaToOneEnable = VK_FALSE; // 선택적
```

나중 장에서 멀티샘플링을 다시 살펴볼 것입니다. 지금은 비활성화된 상태로 두겠습니다.

## 깊이와 스텐실 테스트

깊이 및/또는 스텐실 버퍼를 사용하는 경우, `VkPipelineDepthStencilStateCreateInfo`를 사용하여 깊이 및 스텐실 테스트를 구성해야 합니다. 지금은 없으므로 이러한 구조체에 대한 포인터 대신 `nullptr`을 전달할 수 있습니다. 깊이 버퍼링 장에서 이 부분으로 돌아올 것입니다.

## 색상 블렌딩

프래그먼트 셰이더가 색상을 반환한 후, 이를 이미 프레임버퍼에 있는 색상과 결합해야 합니다. 이 변환을 색상 블렌딩이라고 하며, 이를 수행하는 두 가지 방법이 있습니다:

* 이전 값과 새 값을 혼합하여 최종 색상 생성
* 비트 연산을 사용하여 이전 값과 새 값을 결합

색상 블렌딩을 구성하기 위한 두 가지 구조체가 있습니다. 첫 번째 구조체인 `VkPipelineColorBlendAttachmentState`는 연결된 프레임버퍼별 구성을 포함하고, 두 번째 구조체인 `VkPipelineColorBlendStateCreateInfo`는 *전역* 색상 블렌딩 설정을 포함합니다. 우리의 경우 하나의 프레임버퍼만 있습니다:

```c++
VkPipelineColorBlendAttachmentState colorBlendAttachment{};
colorBlendAttachment.colorWriteMask = VK_COLOR_COMPONENT_R_BIT | VK_COLOR_COMPONENT_G_BIT | VK_COLOR_COMPONENT_B_BIT | VK_COLOR_COMPONENT_A_BIT;
colorBlendAttachment.blendEnable = VK_FALSE;
colorBlendAttachment.srcColorBlendFactor = VK_BLEND_FACTOR_ONE; // 선택적
colorBlendAttachment.dstColorBlendFactor = VK_BLEND_FACTOR_ZERO; // 선택적
colorBlendAttachment.colorBlendOp = VK_BLEND_OP_ADD; // 선택적
colorBlendAttachment.srcAlphaBlendFactor = VK_BLEND_FACTOR_ONE; // 선택적
colorBlendAttachment.dstAlphaBlendFactor = VK_BLEND_FACTOR_ZERO; // 선택적
colorBlendAttachment.alphaBlendOp = VK_BLEND_OP_ADD; // 선택적
```

이 프레임버퍼별 구조체를 통해 첫 번째 색상 블렌딩 방법을 구성할 수 있습니다. 수행될 연산은 다음의 의사 코드로 가장 잘 설명됩니다:

```c++
if (blendEnable) {
    finalColor.rgb = (srcColorBlendFactor * newColor.rgb) <colorBlendOp> (dstColorBlendFactor * oldColor.rgb);
    finalColor.a = (srcAlphaBlendFactor * newColor.a) <alphaBlendOp> (dstAlphaBlendFactor * oldColor.a);
} else {
    finalColor = newColor;
}

finalColor = finalColor & colorWriteMask;
```

`blendEnable`이 `VK_FALSE`로 설정되면, 프래그먼트 셰이더의 새로운 색상이 수정 없이 그대로 전달됩니다. 그렇지 않으면, 새로운 색상을 계산하기 위해 두 가지 혼합 연산이 수행됩니다. 결과 색상은 `colorWriteMask`와 AND 연산되어 실제로 어떤 채널이 전달될지 결정됩니다.

색상 블렌딩을 사용하는 가장 일반적인 방법은 알파 블렌딩을 구현하는 것입니다. 이는 새로운 색상을 그 불투명도를 기반으로 이전 색상과 블렌딩하고자 할 때 사용됩니다. `finalColor`는 다음과 같이 계산되어야 합니다:

```c++
finalColor.rgb = newAlpha * newColor + (1 - newAlpha) * oldColor;
finalColor.a = newAlpha.a;
```

이는 다음과 같은 매개변수들로 구현할 수 있습니다:

```c++
colorBlendAttachment.blendEnable = VK_TRUE;
colorBlendAttachment.srcColorBlendFactor = VK_BLEND_FACTOR_SRC_ALPHA;
colorBlendAttachment.dstColorBlendFactor = VK_BLEND_FACTOR_ONE_MINUS_SRC_ALPHA;
colorBlendAttachment.colorBlendOp = VK_BLEND_OP_ADD;
colorBlendAttachment.srcAlphaBlendFactor = VK_BLEND_FACTOR_ONE;
colorBlendAttachment.dstAlphaBlendFactor = VK_BLEND_FACTOR_ZERO;
colorBlendAttachment.alphaBlendOp = VK_BLEND_OP_ADD;
```

가능한 모든 연산은 사양서의 `VkBlendFactor`와 `VkBlendOp` 열거형에서 찾을 수 있습니다.

두 번째 구조체는 모든 프레임버퍼에 대한 구조체 배열을 참조하며, 앞서 언급한 계산에서 블렌드 팩터로 사용할 수 있는 블렌드 상수를 설정할 수 있게 해줍니다.

```c++
VkPipelineColorBlendStateCreateInfo colorBlending{};
colorBlending.sType = VK_STRUCTURE_TYPE_PIPELINE_COLOR_BLEND_STATE_CREATE_INFO;
colorBlending.logicOpEnable = VK_FALSE;
colorBlending.logicOp = VK_LOGIC_OP_COPY; // 선택적
colorBlending.attachmentCount = 1;
colorBlending.pAttachments = &colorBlendAttachment;
colorBlending.blendConstants[0] = 0.0f; // 선택적
colorBlending.blendConstants[1] = 0.0f; // 선택적
colorBlending.blendConstants[2] = 0.0f; // 선택적
colorBlending.blendConstants[3] = 0.0f; // 선택적
```

두 번째 블렌딩 방법(비트 단위 결합)을 사용하고 싶다면, `logicOpEnable`을 `VK_TRUE`로 설정해야 합니다. 그러면 비트 단위 연산을 `logicOp` 필드에서 지정할 수 있습니다. 이렇게 하면 마치 연결된 모든 프레임버퍼에 대해 `blendEnable`을 `VK_FALSE`로 설정한 것처럼 첫 번째 방법이 자동으로 비활성화된다는 점에 주의하세요! `colorWriteMask`는 이 모드에서도 프레임버퍼의 어떤 채널이 실제로 영향을 받을지 결정하는 데 사용됩니다. 여기서 우리가 한 것처럼 두 모드를 모두 비활성화하는 것도 가능한데, 이 경우 프래그먼트 색상이 수정 없이 프레임버퍼에 기록됩니다.

## 파이프라인 레이아웃

셰이더에서 `uniform` 값을 사용할 수 있는데, 이는 동적 상태 변수와 비슷한 전역 변수로, 셰이더를 재생성하지 않고도 드로잉 시점에 변경하여 셰이더의 동작을 변경할 수 있습니다. 이는 일반적으로 버텍스 셰이더에 변환 행렬을 전달하거나, 프래그먼트 셰이더에서 텍스처 샘플러를 생성하는 데 사용됩니다.

이러한 uniform 값들은 `VkPipelineLayout` 객체를 생성하여 파이프라인 생성 중에 지정되어야 합니다. 나중 장까지 이를 사용하지는 않겠지만, 빈 파이프라인 레이아웃은 생성해야 합니다.

나중에 다른 함수에서 이 객체를 참조할 것이므로, 클래스 멤버로 이 객체를 저장할 변수를 만듭니다:

```c++
VkPipelineLayout pipelineLayout;
```

그리고 `createGraphicsPipeline` 함수에서 객체를 생성합니다:

```c++
VkPipelineLayoutCreateInfo pipelineLayoutInfo{};
pipelineLayoutInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
pipelineLayoutInfo.setLayoutCount = 0; // 선택적
pipelineLayoutInfo.pSetLayouts = nullptr; // 선택적
pipelineLayoutInfo.pushConstantRangeCount = 0; // 선택적
pipelineLayoutInfo.pPushConstantRanges = nullptr; // 선택적

if (vkCreatePipelineLayout(device, &pipelineLayoutInfo, nullptr, &pipelineLayout) != VK_SUCCESS) {
    throw std::runtime_error("failed to create pipeline layout!");
}
```

이 구조체는 또한 *푸시 상수*를 지정하는데, 이는 셰이더에 동적 값을 전달하는 또 다른 방법으로, 나중 장에서 다룰 수 있습니다. 파이프라인 레이아웃은 프로그램의 수명 전체에 걸쳐 참조될 것이므로, 마지막에 파괴해야 합니다:

```c++
void cleanup() {
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    ...
}
```

## 결론

이것으로 모든 고정 함수 상태가 끝났습니다! 이 모든 것을 처음부터 설정하는 것은 많은 작업이 필요하지만, 장점은 이제 그래픽스 파이프라인에서 일어나는 모든 것을 거의 완전히 알게 되었다는 것입니다! 이는 특정 컴포넌트의 기본 상태가 예상과 다른 경우에 발생할 수 있는 예기치 않은 동작의 가능성을 줄여줍니다.

하지만 그래픽스 파이프라인을 마침내 생성하기 전에 생성해야 할 객체가 하나 더 있습니다. 바로 [렌더 패스](!en/Drawing_a_triangle/Graphics_pipeline_basics/Render_passes)입니다.

[C++ 코드](/code/10_fixed_functions.cpp) /
[버텍스 셰이더](/code/09_shader_base.vert) /
[프래그먼트 셰이더](/code/09_shader_base.frag)
