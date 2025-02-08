# 결론

이제 이전 장들에서 다룬 모든 구조체와 객체들을 조합하여 그래픽스 파이프라인을 생성할 수 있습니다! 지금까지 우리가 다룬 객체들의 종류를 간단히 복습해보겠습니다:

* **셰이더 단계(Shader stages)**: 그래픽스 파이프라인의 프로그래밍 가능한 단계의 기능을 정의하는 셰이더 모듈
* **고정 함수 상태(Fixed-function state)**: 입력 어셈블리(input assembly), 래스터라이저(rasterizer), 뷰포트(viewport), 컬러 블렌딩(color blending) 등과 같은 파이프라인의 고정 함수 단계를 정의하는 모든 구조체
* **파이프라인 레이아웃(Pipeline layout)**: 셰이더에서 참조하는 유니폼(uniform) 및 푸시(push) 값으로, 드로우 시간에 업데이트할 수 있음
* **렌더 패스(Render pass)**: 파이프라인 단계에서 참조하는 어태치먼트(attachments)와 그 사용 방법

이 모든 것들이 조합되어 그래픽스 파이프라인의 기능을 완전히 정의합니다. 따라서 이제 `createGraphicsPipeline` 함수의 마지막 부분에서 `VkGraphicsPipelineCreateInfo` 구조체를 채워나갈 수 있습니다. 하지만 이 작업은 `vkDestroyShaderModule` 호출 전에 이루어져야 합니다. 왜냐하면 셰이더 모듈은 파이프라인 생성 과정에서 여전히 사용되기 때문입니다.

```c++
VkGraphicsPipelineCreateInfo pipelineInfo{};
pipelineInfo.sType = VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO;
pipelineInfo.stageCount = 2;
pipelineInfo.pStages = shaderStages;
```

먼저, `VkPipelineShaderStageCreateInfo` 구조체 배열을 참조합니다.

```c++
pipelineInfo.pVertexInputState = &vertexInputInfo;
pipelineInfo.pInputAssemblyState = &inputAssembly;
pipelineInfo.pViewportState = &viewportState;
pipelineInfo.pRasterizationState = &rasterizer;
pipelineInfo.pMultisampleState = &multisampling;
pipelineInfo.pDepthStencilState = nullptr; // 선택적(Optional)
pipelineInfo.pColorBlendState = &colorBlending;
pipelineInfo.pDynamicState = &dynamicState;
```

그 다음, 고정 함수 단계를 설명하는 모든 구조체들을 참조합니다.

```c++
pipelineInfo.layout = pipelineLayout;
```

그 후에는 파이프라인 레이아웃을 참조합니다. 이는 구조체 포인터가 아닌 Vulkan 핸들입니다.

```c++
pipelineInfo.renderPass = renderPass;
pipelineInfo.subpass = 0;
```

마지막으로, 렌더 패스와 이 그래픽스 파이프라인이 사용될 서브패스(subpass)의 인덱스를 참조합니다. 이 특정 인스턴스 대신 다른 렌더 패스를 사용할 수도 있지만, 해당 렌더 패스는 `renderPass`와 *호환*되어야 합니다. 호환성에 대한 요구 사항은 [여기](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap8.html#renderpass-compatibility)에 설명되어 있지만, 이 튜토리얼에서는 해당 기능을 사용하지 않을 것입니다.

```c++
pipelineInfo.basePipelineHandle = VK_NULL_HANDLE; // 선택적(Optional)
pipelineInfo.basePipelineIndex = -1; // 선택적(Optional)
```

실제로 두 개의 추가 매개변수가 더 있습니다: `basePipelineHandle`과 `basePipelineIndex`. Vulkan은 기존 파이프라인에서 파생된 새로운 그래픽스 파이프라인을 생성할 수 있도록 합니다. 파이프라인 파생의 아이디어는 기존 파이프라인과 많은 기능을 공유할 때 파이프라인 설정 비용이 적게 들고, 동일한 부모 파이프라인 간 전환도 더 빠르게 할 수 있다는 것입니다. `basePipelineHandle`을 사용하여 기존 파이프라인의 핸들을 지정하거나, `basePipelineIndex`를 사용하여 생성될 다른 파이프라인을 인덱스로 참조할 수 있습니다. 현재는 단일 파이프라인만 있으므로, null 핸들과 유효하지 않은 인덱스를 지정하겠습니다. 이러한 값들은 `VkGraphicsPipelineCreateInfo`의 `flags` 필드에 `VK_PIPELINE_CREATE_DERIVATIVE_BIT` 플래그가 지정된 경우에만 사용됩니다.

이제 마지막 단계를 준비하기 위해 `VkPipeline` 객체를 보관할 클래스 멤버를 생성합니다:

```c++
VkPipeline graphicsPipeline;
```

그리고 마지막으로 그래픽스 파이프라인을 생성합니다:

```c++
if (vkCreateGraphicsPipelines(device, VK_NULL_HANDLE, 1, &pipelineInfo, nullptr, &graphicsPipeline) != VK_SUCCESS) {
    throw std::runtime_error("failed to create graphics pipeline!");
}
```

`vkCreateGraphicsPipelines` 함수는 일반적인 Vulkan 객체 생성 함수보다 더 많은 매개변수를 가지고 있습니다. 이 함수는 여러 개의 `VkGraphicsPipelineCreateInfo` 객체를 받아 여러 개의 `VkPipeline` 객체를 한 번에 생성하도록 설계되었습니다.

두 번째 매개변수는 `VK_NULL_HANDLE`을 전달한 부분으로, 선택적인 `VkPipelineCache` 객체를 참조합니다. 파이프라인 캐시는 `vkCreateGraphicsPipelines` 호출 간에, 심지어 프로그램 실행 간에도 파이프라인 생성과 관련된 데이터를 저장하고 재사용하는 데 사용될 수 있습니다. 이를 통해 나중에 파이프라인 생성 속도를 크게 높일 수 있습니다. 이에 대해서는 파이프라인 캐시 장에서 더 자세히 다룰 것입니다.

그래픽스 파이프라인은 모든 일반적인 드로우 작업에 필수적이므로, 프로그램 종료 시에만 파괴되어야 합니다:

```c++
void cleanup() {
    vkDestroyPipeline(device, graphicsPipeline, nullptr);
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    ...
}
```

이제 프로그램을 실행하여 이 모든 노력이 성공적인 파이프라인 생성으로 이어졌는지 확인해보세요! 이제 화면에 무언가가 나타나는 것까지 얼마 남지 않았습니다. 다음 몇 장에서는 스왑 체인 이미지로부터 실제 프레임버퍼를 설정하고 드로우 명령을 준비할 것입니다.

[C++ 코드](/code/12_graphics_pipeline_complete.cpp) /
[버텍스 셰이더](/code/09_shader_base.vert) /
[프래그먼트 셰이더](/code/09_shader_base.frag)
