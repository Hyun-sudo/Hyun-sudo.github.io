---
title: "irq_trace_ftrace.sh 파일 echo: write error: Invaild argument 해결 방법"
excerpt: "교재의 실습 코드 오류를 파악하고 해결"

categories:
    - Linux Kernel Debugging
tags:
    - [linux, kernel, debugging, shellscript]

toc: true
toc_sticky: true

date: 2022-02-16
last_modified_at: 2022-02-16
---

## irq_trace_ftrace.sh 실행 오류

저자의 블로그에도 올라간 내용이지만,<br>
이 글에서는 내가 수정한 쉘 스크립트를 같이 첨부하였다.<br>
1판 2쇄 기준으로 86p에 irq_trace_ftrace.sh의 코드와 관련된 내용이다.

커널 빌드, 설치를 한 이후 irq_trace_ftrace.sh를 실행하면<br>
line 19: echo: write error: Invaild argument라는 에러가 뜰 수 있다.

먼저 에러가 발생한 부분의 코드를 살펴보자.

```bash
echo rpi_get_interrupt_info > /sys/kerenl/debug/tracing/set_ftrace_filter
```

83p의 패치 코드에서 작성한 rpi_get_interrupt_info()라는 함수명을<br>
/sys/kernel/debug/tracing/set_ftrace_filter에 지정하는 명령어인데,<br>
에러가 뜨는 이유는 해당 함수가 제대로 빌드/설치되지 않았기 때문이다.

'/sys/kernel/debug/tracing/set_ftrace_filter' 파일에 지정할 수 있는 함수 이름은<br>
'/sys/kernel/debug/trace/available_filter_functions'에 있어야 한다.<br>
해당 디렉토리로 이동해 확인해보자.

![image](/images/available_filter_functions.png){: .align-center}

이제 에디터를 통해 available_filter_functions을 열어 rpi_get_interrupt_info()를 찾아보자.

![image](/images/rpi_get_interrupt_info.png){: .align-center}

이와 같이 나오지 않는다면 커널을 다시 빌드하고 설치해야한다.<br>
교재는 이후 90p에서 bcm2835_mmc_irq() 함수에도 ftrace를 설정하는 코드를 추가하는데<br>
저자의 블로그에서 제시한 수정된 코드를 포함해 아래에 쉘 스크립트 전체를 첨부하겠다.


## 수정된 Shell Script

```bash
#!/bin/bash
 
echo 0 > /sys/kernel/debug/tracing/tracing_on
sleep 1
echo "tracing_off" 
 
echo 0 > /sys/kernel/debug/tracing/events/enable
sleep 1
echo "events disabled"
 
echo  secondary_start_kernel  > /sys/kernel/debug/tracing/set_ftrace_filter	
sleep 1
echo "set_ftrace_filter init"
 
echo function > /sys/kernel/debug/tracing/current_tracer
sleep 1
echo "function tracer enabled"
 
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_switch/enable
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_entry/enable
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_exit/enable
sleep 1
echo "event enabled"
 
echo bcm2835_mmc_irq > /sys/kernel/debug/tracing/set_ftrace_filter
sleep 1
echo "bcm2835_mmc_irq is enabled"
 
echo rpi_get_interrupt_info > /sys/kernel/debug/tracing/set_ftrace_filter
sleep 1
echo "rpi_get_interrupt_info is  enabled"
 
echo 1 > /sys/kernel/debug/tracing/options/func_stack_trace
echo 1 > /sys/kernel/debug/tracing/options/sym-offset
echo "function stack trace enabled"
 
echo 1 > /sys/kernel/debug/tracing/tracing_on
echo "tracing_on"
```

86p의 기존 코드와 다른 점은 기존 코드의 중복되는 명령은 삭제했고<br>
bcm2835_mmc_irq()와 rpi_get_interrupt_info()를 따로 따로 필터에 저장해<br>
어디서 문제가 생겼는지 확인할 수 있게 했다.
