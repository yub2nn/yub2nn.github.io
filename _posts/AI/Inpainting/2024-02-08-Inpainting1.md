---
layout: post
related_posts: 
title: Inpainting 이란?
date: 2024-08-10 11:00:00 +0900
categories: 
    - ai
    - inpainting
---
* * *
* toc
{:toc}

## 1. Inpainting
: 손상 및 누락된 부분을 재구성하는 기술

 ![galaxyAI](https://blog.kakaocdn.net/dn/dy7nqX/btsGCW3Rkjg/OPBg7IbAGKJZAaSj2l7Q21/img.gif) ![](https://blog.kakaocdn.net/dn/TkI7p/btsGFsHsUr3/rkW2KQkdEeeCxO2x1Oklx1/img.jpg)

- **복원, 생성**
    1. 미용적 : AI 지우개
    2. **생산적** : 공장자동화, 탐사로봇, 드론 등의 **빛반사 제거** → ‘자율주행’ 빛반사로 인한 탐지 성능 향상

## 2. Image completion
: Inpainting + Outpainting

(복원 영역이 이미지 안 : In , 밖 : Out)

![Image Outpainting](https://blog.kakaocdn.net/dn/bEmaZM/btsGEkJZ5N6/rvDFKHgoOPW4rDGdvXtzS1/img.png)

- Outpainting은 디스플레이 산업에서 주로 쓰임

## 3. Image Processing vs Inpainting

![Image Processing (Dehazing, Deep-WaveNet)](https://blog.kakaocdn.net/dn/bQUCfv/btsGCv6zAdL/McqlIrbOC1ddZ8xsv8jINk/img.png)

![Image Processing (Super Resolution, SRResNet)](https://blog.kakaocdn.net/dn/bjatzi/btsGE471DKA/26TOygkCrpUhvIpoKkdvx0/img.png)

![Image Inpainting (LAMA)](https://blog.kakaocdn.net/dn/DVszG/btsGFxu9p4e/6KE7LLCvwvkZ5j1WJ4q5CK/img.png)

--- | ---
**Image Processing** (Denoising, Dehazing, Super Resolution) | **Inpainting**
이미지 전체 | **RoI** 존재 = **Masking Image** **Target**을 염두하고 사용 (빛반사 등)

Input Image + ( Input Image → Specular highlight detection ) → Inpainting → Output Image

## 4. Masking Image
![](https://blog.kakaocdn.net/dn/FcIgV/btsGEWbbMJN/qVgEuL2h84mTDky4z8cnIK/img.png)

1. 수동 : 직접 찍는 방법
2. 자동 : Active method (ex. 빛반사 구분 task (Clustering))

## 5. Accuracy & FPS

- **Accuracy**
    1. **P2P (pixel to pixel)  
        : 주변 픽셀을 참고해 가장자리부터 하나씩 채워감 (Past)  
        patch base & diffusion base  
        Image Jitter 존재 (현실감 감소)
    2. **Generative model  
        : Dilated → (Contextual + Dilated) → Output → Global / Local Critics  
        팽창 후 밑그림을 그려서 주변 픽셀과 비교해 병렬 구조로 한번에 output (Nowadays)  
        Hallucination risk**  
        

	-  **GANs 구조  
        Local Discriminator → Masking 영역 판별  
        Global Discriminator → 이미지 자체 판별
    * **CNN 구조  
        Convolution → 주변 픽셀 이용해 연산x

> **Dilated 하는 이유 ?**  
> masking의 가장자리가 채워지지 않고 테두리가 남는 현상을 최소화하기 위해 팽창시켜 처리

- **FPS**
![Video Inpainting (E2FGVI)](https://blog.kakaocdn.net/dn/dl3NHx/btsGFE17HrY/m56keJpP0ktW8jjY5KLKR0/img.png)
1. **Video Inpainting  
    : HW의 발전 + Neural Nets 로 인해 빨라짐  
    - **Sampling Period
        시간 간격 기준 (0.01 ~ 0.001초 사이) 적절히 선정  
        너무 짧으면 영상이 변하지 않는 것으로 인식  
        너무 길면 자연스럽게 처리되지 않음

---
## Reference

[https://r1.community.samsung.com/t5/갤럭시-s/갤럭시-꿀-tip-포토에디터-ai-지우개-디테일하게-활용하기/td-p/22316142](https://r1.community.samsung.com/t5/%EA%B0%A4%EB%9F%AD%EC%8B%9C-s/%EA%B0%A4%EB%9F%AD%EC%8B%9C-%EA%BF%80-tip-%ED%8F%AC%ED%86%A0%EC%97%90%EB%94%94%ED%84%B0-ai-%EC%A7%80%EC%9A%B0%EA%B0%9C-%EB%94%94%ED%85%8C%EC%9D%BC%ED%95%98%EA%B2%8C-%ED%99%9C%EC%9A%A9%ED%95%98%EA%B8%B0/td-p/22316142)

[https://lensvid.com/gear/google-and-mit-demonstrate-reflection-removal-technology-from-images/](https://lensvid.com/gear/google-and-mit-demonstrate-reflection-removal-technology-from-images/)

[https://github.com/ShinyCode/image-outpainting](https://github.com/ShinyCode/image-outpainting)

[https://github.com/pksvision/Deep-WaveNet-Underwater-Image-Restoration](https://github.com/pksvision/Deep-WaveNet-Underwater-Image-Restoration)

[https://arxiv.org/pdf/2109.07161.pdf](https://arxiv.org/pdf/2109.07161.pdf)

[https://arxiv.org/pdf/1609.04802.pdf](https://arxiv.org/pdf/1609.04802.pdf)

[https://arxiv.org/pdf/2204.02663v2.pdf](https://arxiv.org/pdf/2204.02663v2.pdf)