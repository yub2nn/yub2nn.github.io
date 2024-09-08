---
layout: post
related_posts: 
title: "[FPGA] Stopwatch 설계 (VerilogHDL)"
date: 2024-07-11 11:00:00 +0900
categories:
  - semicon
  - verilog
---
* * *
* toc
{:toc}

vivado 2019.2와 basys3 FPGA board를 이용한 Stopwatch 설계를 진행하였다.

## 1. 설계 사양

설계 사양을 구체적으로 구상해 보았다.

- btn 총 4개를 이용하여 rst, clear, display_mode, run_mode를 설정한다.
- Reset 상태에서 00.00이 display되고, display_mode는 sec. under_sec로 설정된다.
- Run_mode에서 시간 증가, 정지 모드 두 가지 상태가 있으며 btnr에 의해 toggle 된다.
- Clr는 run_mode가 정지 상태일 때만 유효하며 btnu에 의해 display된 내용이 모두 clear 된다.
- Display_mode는 sec. under_sec / min. sec. 두 가지 상태가 있으며 btnd에 의해 toggle 된다.

usec, sec, min은 system clock 인 100MHz를 분주하여 사용하도록 한다.

![](https://blog.kakaocdn.net/dn/mRRRz/btsIpiSmBL3/V4oTOEkKwSDWKhaovmKCAk/img.jpg)
![](https://blog.kakaocdn.net/dn/bqLRix/btsIqP2dOCg/QmYf9K7wbq1hKK64UonEcK/img.jpg)
