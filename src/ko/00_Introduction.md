# 소개

## 안내

이 튜토리얼에서는 [Vulkan](https://www.khronos.org/vulkan/) 그래픽스 및 컴퓨트 API의 기초를 배우게 됩니다. Vulkan은 [Khronos 그룹](https://www.khronos.org/)(OpenGL로 유명한)이 만든 새로운 API로, 현대적인 그래픽 카드를 더 잘 추상화했습니다. 이 새로운 인터페이스를 통해 애플리케이션이 하고자 하는 일을 더 정확하게 설명할 수 있어, [OpenGL](https://en.wikipedia.org/wiki/OpenGL)이나 [Direct3D](https://en.wikipedia.org/wiki/Direct3D) 같은 기존 API들과 비교했을 때 더 나은 성능과 예측 가능한 드라이버 동작을 얻을 수 있습니다. Vulkan의 기본 개념은 [Direct3D 12](https://en.wikipedia.org/wiki/Direct3D#Direct3D_12)와 [Metal](https://en.wikipedia.org/wiki/Metal_(API))과 비슷하지만, Vulkan은 완전한 크로스 플랫폼이며 Windows, Linux, Android에서 동시에 개발할 수 있다는 장점이 있습니다.

하지만 이러한 이점들을 얻기 위해서는 훨씬 더 상세한 API를 다뤄야 한다는 대가를 치러야 합니다. 초기 프레임 버퍼 생성부터 버퍼나 텍스처 이미지 같은 객체들의 메모리 관리까지, 그래픽스 API와 관련된 모든 세부 사항을 애플리케이션에서 처음부터 설정해야 합니다. 그래픽스 드라이버가 도와주는 부분이 훨씬 적어지므로, 올바른 동작을 보장하기 위해 애플리케이션에서 더 많은 작업을 해야 합니다.

여기서 얻을 수 있는 교훈은 Vulkan이 모든 사람을 위한 것은 아니라는 점입니다. 고성능 컴퓨터 그래픽스에 열정적이고 그만큼의 노력을 기꺼이 투자할 의향이 있는 프로그래머들을 대상으로 합니다. 만약 컴퓨터 그래픽스보다 게임 개발에 더 관심이 있다면, OpenGL이나 Direct3D를 계속 사용하는 것이 좋을 수 있습니다. 이들은 당분간 Vulkan을 위해 deprecated될 일이 없을 것입니다. 다른 대안으로는 [Unreal Engine](https://en.wikipedia.org/wiki/Unreal_Engine#Unreal_Engine_4)이나 [Unity](https://en.wikipedia.org/wiki/Unity_(game_engine)) 같은 엔진을 사용하는 것입니다. 이러한 엔진들은 Vulkan을 내부적으로 사용하면서도 더 높은 수준의 API를 제공합니다.

이제 이 튜토리얼을 따라하기 위한 몇 가지 전제 조건을 살펴보겠습니다:

* Vulkan을 지원하는 그래픽 카드와 드라이버 ([NVIDIA](https://developer.nvidia.com/vulkan-driver), [AMD](http://www.amd.com/en-us/innovations/software-technologies/technologies-gaming/vulkan), [Intel](https://software.intel.com/en-us/blogs/2016/03/14/new-intel-vulkan-beta-1540204404-graphics-driver-for-windows-78110-1540), [Apple Silicon (또는 Apple M1)](https://www.phoronix.com/scan.php?page=news_item&px=Apple-Silicon-Vulkan-MoltenVK))
* C++ 경험 (RAII, 초기화 리스트에 대한 친숙도)
* C++17 기능을 제대로 지원하는 컴파일러 (Visual Studio 2017+, GCC 7+, 또는 Clang 5+)
* 3D 컴퓨터 그래픽스에 대한 기본 지식

이 튜토리얼에서는 OpenGL이나 Direct3D 개념에 대한 지식을 전제로 하지는 않지만, 3D 컴퓨터 그래픽스의 기초는 알고 있어야 합니다. 예를 들어 원근 투영에 대한 수학적 설명은 하지 않을 것입니다. 컴퓨터 그래픽스 개념에 대한 훌륭한 소개는 [이 온라인 책](https://paroj.github.io/gltut/)을 참고하세요. 다른 훌륭한 컴퓨터 그래픽스 리소스들은 다음과 같습니다:

* [Ray tracing in one weekend](https://github.com/RayTracing/raytracing.github.io)
* [Physically Based Rendering book](http://www.pbr-book.org/)
* 실제 엔진에서 Vulkan이 사용된 오픈소스 [Quake](https://github.com/Novum/vkQuake)와 [DOOM 3](https://github.com/DustinHLand/vkDOOM3)

C++ 대신 C를 사용할 수도 있지만, 다른 선형 대수 라이브러리를 사용해야 하고 코드 구조화는 직접 해야 합니다. 우리는 로직과 리소스 수명을 관리하기 위해 클래스와 RAII 같은 C++ 기능들을 사용할 것입니다. Rust 개발자들을 위한 두 가지 대체 버전의 튜토리얼도 있습니다: [Vulkano 기반](https://github.com/bwasty/vulkan-tutorial-rs), [Vulkanalia 기반](https://kylemayes.github.io/vulkanalia).

다른 프로그래밍 언어를 사용하는 개발자들이 쉽게 따라할 수 있도록, 그리고 기본 API를 경험하기 위해 우리는 Vulkan과 작업할 때 원래의 C API를 사용할 것입니다. 하지만 C++를 사용하고 있다면, 일부 지저분한 작업을 추상화하고 특정 유형의 오류를 방지하는 데 도움이 되는 새로운 [Vulkan-Hpp](https://github.com/KhronosGroup/Vulkan-Hpp) 바인딩을 사용하는 것이 좋을 수 있습니다.

## E-book

이 튜토리얼을 e-book으로 읽고 싶다면 EPUB나 PDF 버전을 다운로드할 수 있습니다:

* [EPUB](https://vulkan-tutorial.com/resources/vulkan_tutorial_en.epub)
* [PDF](https://vulkan-tutorial.com/resources/vulkan_tutorial_en.pdf)

## 튜토리얼 구조

우리는 Vulkan이 어떻게 작동하는지, 그리고 화면에 첫 삼각형을 그리기 위해 해야 할 작업들에 대한 개요부터 시작할 것입니다. 전체적인 그림 속에서 각각의 작은 단계들이 어떤 역할을 하는지 이해하고 나면, 그 단계들의 목적이 더 명확해질 것입니다. 다음으로 [Vulkan SDK](https://lunarg.com/vulkan-sdk/), 선형 대수 연산을 위한 [GLM 라이브러리](http://glm.g-truc.net/), 윈도우 생성을 위한 [GLFW](http://www.glfw.org/)로 개발 환경을 설정할 것입니다. 이 튜토리얼에서는 Visual Studio를 사용하는 Windows와 GCC를 사용하는 Ubuntu Linux에서의 설정 방법을 다룰 것입니다.

그 다음에는 첫 삼각형을 렌더링하는 데 필요한 Vulkan 프로그램의 모든 기본 구성 요소들을 구현할 것입니다. 각 챕터는 대략 다음과 같은 구조를 따릅니다:

* 새로운 개념과 그 목적 소개
* 프로그램에 통합하기 위한 모든 관련 API 호출 사용
* 일부를 헬퍼 함수로 추상화

각 챕터는 이전 챕터의 후속편으로 작성되었지만, 특정 Vulkan 기능을 소개하는 독립적인 글로도 읽을 수 있습니다. 즉, 이 사이트는 참조 자료로도 유용합니다. 모든 Vulkan 함수와 타입은 명세서에 링크되어 있어서 클릭하면 더 자세히 알아볼 수 있습니다. Vulkan은 매우 새로운 API이므로 명세서 자체에 일부 부족한 점이 있을 수 있습니다. [이 Khronos 저장소](https://github.com/KhronosGroup/Vulkan-Docs)에 피드백을 제출하시기 바랍니다.

앞서 언급했듯이, Vulkan API는 그래픽스 하드웨어를 최대한 제어할 수 있도록 많은 매개변수를 가진 상당히 자세한 API를 가지고 있습니다. 이로 인해 텍스처 생성과 같은 기본적인 작업도 매번 반복해야 하는 많은 단계가 필요합니다. 따라서 우리는 튜토리얼 전반에 걸쳐 자체적인 헬퍼 함수 모음을 만들 것입니다.

각 챕터는 또한 그 시점까지의 전체 코드 링크로 마무리됩니다. 코드의 구조에 대해 의문이 있거나 버그가 있어서 비교해보고 싶을 때 참조할 수 있습니다. 모든 코드 파일은 정확성을 검증하기 위해 여러 벤더의 그래픽 카드에서 테스트되었습니다. 각 챕터 끝에는 해당 주제와 관련된 질문을 할 수 있는 댓글 섹션도 있습니다. 도움을 받기 위해서는 플랫폼, 드라이버 버전, 소스 코드, 예상 동작, 실제 동작을 명시해 주시기 바랍니다.

이 튜토리얼은 커뮤니티의 노력으로 만들어지는 것을 목표로 합니다. Vulkan은 아직 매우 새로운 API이며 모범 사례가 아직 제대로 확립되지 않았습니다. 튜토리얼과 사이트 자체에 대한 어떤 종류의 피드백이라도 있다면 [GitHub 저장소](https://github.com/Overv/VulkanTutorial)에 이슈나 풀 리퀘스트를 제출해 주시기 바랍니다. 저장소를 *watch*하면 튜토리얼 업데이트 알림을 받을 수 있습니다.

화면에 첫 Vulkan 삼각형을 그리는 의식을 마친 후에는, 선형 변환, 텍스처, 3D 모델을 포함하도록 프로그램을 확장할 것입니다.

이전에 그래픽스 API들을 다뤄보셨다면, 화면에 첫 기하도형이 나타날 때까지 많은 단계가 필요하다는 것을 아실 것입니다. Vulkan에도 이러한 초기 단계들이 많이 있지만, 각각의 개별 단계들이 이해하기 쉽고 불필요하게 느껴지지 않는다는 것을 보게 될 것입니다. 또한 지루해 보이는 삼각형을 그리고 나면, 완전히 텍스처가 입혀진 3D 모델을 그리는 데는 그렇게 많은 추가 작업이 필요하지 않고, 그 이후의 각 단계는 훨씬 더 보람있다는 점을 기억하는 것이 중요합니다.

튜토리얼을 따라하다가 문제가 발생하면, 먼저 FAQ를 확인하여 문제와 해결책이 이미 나열되어 있는지 확인하세요. 그래도 해결되지 않는다면 가장 관련된 챕터의 댓글 섹션에서 자유롭게 도움을 요청하세요.

고성능 그래픽스 API의 미래로 뛰어들 준비가 되셨나요? [시작해봅시다!](!ko/Overview)

## 라이선스

Copyright (C) 2015-2023, Alexander Overvoorde

달리 명시되지 않는 한 이 콘텐츠는 [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) 라이선스 하에 제공됩니다.
기여하면서, 귀하는 동일한 라이선스 하에 귀하의 기여를 공개적으로 라이선스하는 것에 동의합니다.

소스 저장소의 `code` 디렉토리에 있는 코드 목록은 [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/) 하에 라이선스됩니다.
해당 디렉토리에 기여하면서, 귀하는 동일한 퍼블릭 도메인과 같은 라이선스 하에 귀하의 기여를 공개적으로 라이선스하는 것에 동의합니다.

이 프로그램은 유용하게 사용될 수 있기를 바라며 배포되지만, 특정 목적에 대한 적합성이나 상품성에 대한 보증을 포함한 어떠한 보증도 제공하지 않습니다.