# Vulkan Tutorial (Korean Translation)

This repository hosts a Korean translation of [Vulkan Tutorial](https://vulkan-tutorial.com/) by **Alexander Overvoorde**. The original tutorial was structured using daux.io, but it has been converted to `mdbook` format for easier maintenance and deployment. The translation was assisted by ChatGPT, Claude, and DeepSeek.

I am continuously improving the Korean translation while studying Vulkan, and I plan to refine it until it reaches a satisfactory level. Once I am confident in the quality, I intend to request official support for this translation in the VulkanTutorial repository.

## Deployment
- [https://sanggukim.github.io/VulkanTutorial/](https://sanggukim.github.io/VulkanTutorial/)

## Purpose
This project was created for personal learning purposes and has been publicly shared for others who may find it useful.

## Repository Structure
- `src/` : Contains the original documentation in `mdbook` format.
- `.github/` : Contains GitHub Actions configurations for deployment.

## Building Locally
To build and serve the translated tutorial locally, follow these steps:
```sh
# Install mdbook
cargo install mdbook

# Serve the book locally
mdbook serve
```
Then, open `http://localhost:3000` in your browser to view the tutorial.

## License
The contents of this repository are licensed under **CC BY-SA 4.0**, unless stated otherwise. By contributing to this repository, you agree to license your contributions under the same license.

The code listings in the tutorial are licensed under **CC0 1.0 Universal**. By contributing to the repository, you agree to license your contributions to the public under this public domain-like license.

---

# Vulkan 튜토리얼 (한국어 번역)

이 저장소는 **Alexander Overvoorde**가 작성한 [Vulkan Tutorial](https://vulkan-tutorial.com/)의 한국어 번역본을 제공합니다. 원본 튜토리얼은 daux.io를 기반으로 제작되었으나, 유지보수 및 배포 편의를 위해 `mdbook` 형식으로 변환되었습니다. 번역 과정에서는 ChatGPT, Claude, DeepSeek의 도움을 받았습니다.

저는 직접 Vulkan을 학습하며 한국어 번역 품질을 지속적으로 개선할 예정이며, 만족할 만한 수준에 도달하면 VulkanTutorial 저장소에서 공식적으로 지원받을 수 있도록 요청할 계획입니다.

## 배포 주소
- [https://sanggukim.github.io/VulkanTutorial/](https://sanggukim.github.io/VulkanTutorial/)

## 목적
이 프로젝트는 개인적인 학습을 목적으로 제작되었으며, Vulkan을 배우는 다른 분들에게도 도움이 될 수 있도록 공개하였습니다.

## 저장소 구조
- `src/` : `mdbook` 형식으로 변환된 문서가 포함되어 있습니다.
- `.github/` : GitHub Actions 배포 설정 파일이 포함되어 있습니다.

## 로컬 빌드 방법
다음 단계를 따라 번역된 튜토리얼을 로컬에서 빌드하고 실행할 수 있습니다:
```sh
# mdbook 설치
cargo install mdbook

# 로컬 서버 실행
mdbook serve
```
이후 브라우저에서 `http://localhost:3000`을 열어 튜토리얼을 확인할 수 있습니다.

## 라이선스
이 저장소의 콘텐츠는 특별한 언급이 없는 한 **CC BY-SA 4.0** 라이선스를 따릅니다. 이 저장소에 기여하는 경우, 동일한 라이선스하에 기여 내용을 공개하는 것에 동의하는 것으로 간주됩니다.

튜토리얼에 포함된 코드 목록은 **CC0 1.0 Universal** 라이선스를 따릅니다. 따라서 이 저장소에 기여하는 경우, 해당 코드가 공공 도메인과 유사한 라이선스로 공개되는 것에 동의하는 것으로 간주됩니다.
