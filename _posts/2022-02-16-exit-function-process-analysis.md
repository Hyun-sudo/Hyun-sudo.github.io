---
title: "4.4.2 exit() 함수로 프로세스가 종료되는 과정 및 ftrace 로그 분석"
excerpt: "유저 프로세스가 POSIX exit 시스템 콜을 발생시켰을 때 커널 내부에서 프로세스가 종료되는 흐름 파악"

categories:
    - Linux Kernel Debugging
tags:
    - [linux, kernel, debugging, process, ftrace]

toc: true
toc_sticky: true

date: 2022-02-16
last_modified_at: 2022-02-16
---
## ftrace 메세지를 활용한 프로세스 생성과 종료 과정 분석-생성

![image](/images/copy_process_part.png){:   align-center}

프로세스 생성 단계의 ftrace log 이다.<br>
'bash-13081'을 통해 33834~33840번째 ftrace 로그를 출력하는 실체가<br>
pid가 13081인 bash 프로세스라는 사실을 알 수 있다.<br>
또한 pid가 13081인 bash 프로세스가 rpi_proc_exit 프로세스를 생성하는 흐름이다.<br>
sys_clone() 함수를 통해 _do_fork() 함수를 호출하기 때문이다.

![image](/images/rpi_proc_exit.png){:   align-center}

유저 공간에서 exit() 함수를 호출하면<br>
커널 공간에서 실행되는 sys_exit_group() 함수가 호출된다.<br>

이전 절에서는 종료 시그널(sig=9)을 받아 커널 do_exit() 함수가 호출됐다.<br>
그런데 이번에는 유저 공간에서 exit() 함수를 호출해<br>
커널 공간에서 해당 시스템 콜 핸들러인 sys_exit_group() 함수로 시작해 do_exit() 함수가 호출된다.<br>
즉, 프로세스가 종료될 때는 커널의 do_exit() 함수가 호출된다.

## sys_exit_group() 함수가 호출됐다고 판단할 수 있는 근거

ftrace 로그에서 함수의 이름을 보면<br>
함수의 심벌 이름과 함수가 호출되는 코드의 함수 오프셋 주소가 출력된다.

```bash
do_group_exit+0x50/0xe8
```

이것은 do_group_exit() 함수 시작 주소를 기준으로 0x50 떨어진 코드를 의미한다.<br>
그런데 ARMv7 프로세서는 파이프라인을 적용한 아키텍처라서<br>
실제로 호출되는 주소보다 +0x4 바이트 오프셋을 출력한다.

0x50 - 0x4 = 0x4c 이므로<br>
do_group_exit+0x50의 실제 코드 오프셋은 do_group_exit+0x4c이다.

그런데 26034번 메세지를 보면 __wake_up_parent 함수의 시작 주소를 기준으로 0x0만큼 떨어진 주소 정보를 출력한다.

```bash
=> do_group_exit+0x50/0xe8
=> __wake_up_parent+0x0/0x34
=> ret_fast_syscall+0x0/0x28
```

objdump 바이너리 유틸리티를 참고해서 해당 어셈블리 코드를 보자<br>
방법이 기억나지 않는다면 교재의 2.4절을 참고하자.

![image](/images/wake_up_parent.png){:    align-center}

![image](/images/objdump__wake_up_parent.png){:    align-center}

__wake_up_parent() 함수의 시작 주소에서 -0x4 바이트에 있는 어셈블리 코드를 보면<br>
sys_exit_group() 함수의 코드가 보이는데, do_group_exit() 함수를 호출하는 것이다.<br>
따라서 '__wake_up_parent+0x0/0x34' 메시지로 실제 동작한 코드는 'sys_exit_group+0x1c/0x1c'이다.<br>

즉, '__wake_up_parent+0x0' 주소에서 -0x4 바이트 만큼 떨어진 주소에서 sys_exit_group() 함수의 코드가 있는 것이다.<br>
따라서 함수 정보는 다음과 같이 바꿀 수 있다.<br>

```bash
// 기존
=> do_group_exit+0x50/0xe8
=> __wake_up_parent+0x0/0x34
=> ret_fast_syscall+0x0/0x28
 
// 수정
=> do_group_exit+0x4c/0xe4
=> sys_exit_group+0x1c/0x1c
=> ret_fast_syscall+0x0/0x28
```

만약 ftrace 로그에서 함수의 오프셋이 0x0으로 보이면 함수 이름이 제대로 출력되는지 확인하자.

## ftrace 메세지를 활용한 프로세스 생성과 종료 과정 분석-종료

![image](/images/rpi_proc_exit_process_exit.png){:    align-center}

26038번 메세지를 분석해보자.

```bash
rpi_proc_exit-13081 [002] d... 439862.582574: signal_generate: sig=17 errno=0 code=1 comm=bash pid=13081 grp=1 res=0
```

rpi_proc_exit 프로세스는 종료 과정에서 pid가 13081인 부모 bash 프로세스에게 SIGCHLD 시그널을 보낸다.<br>
26038번 메세지는 ftrace의 signal_generate 이벤트로 시그널이 생성됐음을 알린다.<br>
이때 보낸 SIGCHLD 시그널은 'sig=17'이고<br>
'comm=bash pid=13081'이 시그널을 전달 받을 프로세스가 pid가 13081인 bash 프로세스라는 뜻이다.

이처럼 프로세스는 종료될 때 부모 프로세스에게 SIGCHLD 시그널을 보내 자신이 소멸할 것이라는 정보를 알린다.

26041번 메세지를 분석해보자.

```bash
bash-13081 [002] d... 439862.582818: signal_deliver: sig=17 errno=0 code=1 sa_handler=57dc0 sa_flags=14000000
```

이 메세지를 출력하는 주체는 pid가 13081인 bash 프로세스임을 알 수 있다.

- signal_deliver: 시그널을 전달 받음
- sig=17: 전달 받은 시그널
- sa_handler=55a6c: 시그널 핸들러 함수 주소(유저 공간)

이 실습을 통해 다음과 같은 내용을 배웠다.

- 프로세스가 스스로 exit POSIX 시스템 콜을 호출하면 스스로 소멸할 수 있다.
- exit POSIX 시스템 콜에 대한 시스템 콜 핸들러는 sys_exit_group() 함수다.
- 프로세스는 소멸되는 과정에서 부모 프로세스에게 SIGCHLD 시그널을 전달해 자신이 종료될 것이라고 알려준다.