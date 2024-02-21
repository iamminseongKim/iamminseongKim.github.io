---
title: FFMPEG - crf 옵션 별 화질 테스트
aliases: 
tags:
  - 인코딩
  - test
categories:
  - etc
toc: true
toc_label: 목차
date: 2024-02-21
last_modified_at: 2024-02-21
---
회사에서 FFMPEG로 인코딩할 일이 많다 보니 이번 기회에 실험 하려고 한다. 
오늘은 FFMPEG로 인코딩하면서 -crf 옵션에 따른 화질 및 용량을 테스트 해보겠다.
- 환경 : 윈도우 11 
- CPU : 11th Gen Intel(R) Core(TM) i5-1155G7 @ 2.50GHz   2.50 GHz
- RAM : 16.0GB
- GPU : Intel(R) Iris(R) Xe Graphics
- FFMPEG : ffmpeg version 2021-07-11-git-79ebdbb9b9-essentials_build-www.gyan.dev
- 하드웨어 가속은 `-c:v h264_qsv` 사용 시 비트레이트가 너무 떨어져서 사용 안하고 `libx264` 사용

```bash
.\ffmpeg.exe -v error -stats -i "test.mp4" -threads 4 -c:v libx264 -crf 23 -preset medium -r 23.98 -g 48 -keyint_min 48 -profile:v baseline -level 3.0 -pix_fmt yuv420p -vf "scale=1920:1080[s]" -aspect 1.7778 -c:a aac -b:a 192k -ac 2 -ar 44100 -af "volume=1.0" testResult.mp4
```

인코딩 시 사용할 명령어.

### 옵션 설명 
|**옵션** |**설명** |
|---|---|
|-v error|ffmpeg의 출력 레벨을 에러만 출력하도록 설정합니다.|
|-stats|인코딩 통계를 출력합니다.|
|-i "test.mp4"|인코딩할 원본 파일을 지정합니다.|
|-threads 4|인코딩에 사용할 스레드 수를 4개로 설정합니다.|
|-c:v libx264 |비디오 코덱을 H.264로 설정합니다. |
|<font color="#92d050">-crf 23</font>|비디오 코덱의 품질을 나타내는 CRF(Constrained Rate Factor) 값을 23으로 설정합니다.|
|-preset medium |인코딩 속도 프리셋, deafult 값  |
|-r 23.98|비디오의 초당 프레임 수(fps)를 23.98로 설정합니다.|
|-g 48|GOP(Group of Pictures)의 크기를 48프레임으로 설정합니다.|
|-keyint_min 48|키 프레임 간격을 48프레임으로 설정합니다.|
|-profile:v baseline|비디오 코덱의 프로필을 baseline으로 설정합니다.|
|-level 3.0|비디오 코덱의 레벨을 3.0으로 설정합니다.|
|-pix_fmt yuv420p|비디오의 픽셀 형식을 YUV420P로 설정합니다.|
|<font color="#92d050">-vf "scale=1920:1080[s]"</font> |비디오 필터를 설정합니다. eq 필터는 색상 보정을 위한 필터이며, scale 필터는 해상도를 1920x1080으로 변경합니다.|
|-aspect 1.7778|종횡비를 1.7778로 설정합니다.|
|-c:a aac|오디오 코덱을 AAC로 설정합니다.|
|-b:a 192k|오디오 비트레이트를 192kbps로 설정합니다.|
|-ac 2|오디오 채널 수를 2개로 설정합니다.|
|-ar 44100|오디오 샘플링 레이트를 44100Hz로 설정합니다.|
|-af "volume=1.0"|오디오 필터를 설정합니다. volume 필터는 오디오의 음량을 1.0으로 설정합니다.|
|test2Result.mp4|인코딩 결과 파일의 이름을 test2Result.mp4로 설정합니다.|

### CRF 옵션

[ffmpeg wiki](https://trac.ffmpeg.org/wiki/Encode/H.264#crf)

- CRF 스케일의 범위는 0-51
- 0은 무손실(8비트만 해당, 10비트는 -qp 0 사용)
- 23은 기본값
- 51은 가능한 최악의 품질
- 일반적으로 낮은 값이 더 높은 품질
- 적당한 범위는 17 ~ 28 
	- 17 ~ 18 이 시각적으로 무손실에 가까움
- 범위는 지수 함수 적이므로 CRF값을 +6 증가 시키면 대략 **비트 레이트**가 절반으로 줄어들고, -6 이면 두 배가 됨.

### 테스트할 영상 정보

```bash
.\ffprobe.exe .\test.mp4 -show_streams
```

![](https://i.imgur.com/JW7cDNd.png)

- 해상도 : 1920x1080
- bitrate : 4975 kb/s
- codec :  h264
- Duration: 31s
- 용량 : 18.5MB

> 영상 출처 : [pixabay 저작권 없는 영상](https://pixabay.com/ko/videos/%EA%B0%80%EC%A7%9C%EC%9D%98-%EC%A4%80%EB%B9%84-%EC%9E%AC%ED%95%99%EC%83%9D-6015/)


### 테스트 조건 

- 프리셋 변경 없이 `crf`값 만 조정
- `crf` 는 `18`, `23`, `28`, `32` 4가지 비교
- 해상도는 `FHD`, `HD`, `SD` 3가지로 인코딩 
	- scale=1920:1080[s], scale=1280:720[s], scale=854:480[s]
	- ffmpeg 인코딩 시  `-vf `명령어에 넣을 것
- 나머지 세팅은 **동일** (오디오, 필터 등)
- 총 결과는 12개


> 참고 : 윈도우 PowerShell 에서 특정 명령어의 **시간을 측정**하는 방법은 `measure-command { 명령어 }` 이다.

### 테스트 진행

#### 1. FHD (1920x1080)

![](https://i.imgur.com/UsoKbjC.png){: .align-center}
![](https://i.imgur.com/DpoKzGB.png){: .align-center}

원본 영상.

![](https://i.imgur.com/oVaxTqz.png){: .align-center}

`crf : 32` 인 영상

일단 어설프게 18로 하면 안될 것 같다. 비트레이트가 원본 영상을 뛰어 넘어서 용량이 오히려 커져 버렸다.

`23`은 원본이랑 거의 차이가 없고, 압축률은 35퍼센트 정도 나왔다.
`28`은 `약간의 화질`이 떨어진 게 눈에 보이고, 압축률은 71퍼센트가 나왔다. 
`32`는 `확실히 화질`이 떨어진 게 눈에 보이고, 압축률은 83퍼센트가 나왔다. 

> 결론 : **FHD 인코딩** 시 원본과 가장 비슷하게 하려면 `crf 23`
> 저장 효율을 높이고 싶으면 `crf 28` 쪽이 나아보인다.
#### 2. HD (1280x720)

![](https://i.imgur.com/OjHHBcM.png){: .align-center}

![](https://i.imgur.com/esvpxkM.png){: .align-center}

`HD crf 18` 사진인데, 링크 들어가야 정확히 볼 수 있을 듯 함.

HD에서는 솔직히 crf 값 18, 23, 28의 차이를 내 눈으로는 잘 모르겠다.  32는 약간 난다.

> 결론 : 화질을 고려하면 `crf 28`
> 용량을 고려하면 `crf 32`를 쓸 거 같다.

#### 3. SD (854x480)

![](https://i.imgur.com/n3I7Qd8.png){: .align-center}

![](https://i.imgur.com/PyKzOPl.png){: .align-center}

`SD crf 32` 사진. 

여기도 HD와 마찬가지로 화면이 작아지다 보니까 18,23,28 차이는 거의 느껴지지 않고
32 정도만 약간 화질 저하가 보인다. 

> 결론 : 화질을 고려하면 `crf 28`
> 용량을 고려하면 `crf 32`를 쓸 거 같다.