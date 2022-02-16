---
title: "라즈비안 스크린샷 저장 위치 바꾸기"
excerpt: "라즈비안의 기본 스크린샷 저장 위치를 바꿔보자"

categories:
    - Raspbian
tags:
    - [linux, raspberrypi, raspbian, scrot]

toc: true
toc_sticky: true

date: 2022-02-16
last_modified_at: 2022-02-16
---
## 라즈비안에서 현재 화면을 캡쳐하는 법

scrot이 설치되어 있다면(아마 기본으로 설치되어 있을 것이다)<br>
PrtSc 키를 누르거나<br>
터미널에 scrot 명령을 통해 현재 화면을 png 파일로 저장할 수 있다.<br>
그런데 기본 저장 위치가 '/home/pi'로 지정되어 있고<br>
내가 디렉토리를 지정하려면 터미널 명령어를 통해 일일히 지정해야한다.<br>
이 글에서는 키보드 숏컷의 커맨드를 변경해 PrtSc를 누르면 지정 디렉토리에 저장되도록 할 것이다.

## scrot 키보드 숏컷 수정
터미널에서 디렉토리를 지정해 화면 캡쳐를 하는 명령은 다음과 같다.
```bash
scrot /home/pi/Pictures/screenshot.png
```
만약 키보드 숏컷의 커맨드를 이 명령어로 바꾼다면<br>
항상 내가 지정한 디렉토리에 저장될 것이다.<br>
필자는 '/home/pi/Prictures/Screenshots'에 지정할 것이므로 미리 디렉토리를 만들어두었다.<br>
키보드 숏컷은 Openbox의 lxde-pi-rc.xml에서 지정할 수 있는데 에디터를 통해 수정해보자.
```bash
vim /etc/xdg/openbox/lxde-pi-rc.xml
```
![image](/images/scrot_shortcut_change.png){:   .align-center}


lxde-pi-rc.xml 파일에서 PrtSc의 부분을 찾았으면 커맨드를 바꿔주자.<br>
필자는 다음과 같이 바꿨다.
```bash
scrot /home/pi/Pictures/Screenshots/%b%d::%H%M%S.png
```

저장되는 파일의 이름을 '월 일 :: 시 분 초.png'로 지정했다.<br>
키보드 숏컷의 커맨드 부분을 수정했으면 저장하고 리부트를 하자.<br>
이제 PrtSc 버튼을 누르면 내가 지정한 디렉토리에 저장될 것이다.