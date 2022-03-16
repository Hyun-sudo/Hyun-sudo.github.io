---
title: "4.7 프로세스의 종료 과정 분석"
excerpt: "프로세스의 종료 흐름 파악"

categories:
    - Linux Kernel Debugging
tags:
    - [linux, kernel, debugging, process, exit]

toc: true
toc_sticky: true

date: 2022-03-10
last_modified_at: 2022-03-10
---
프로세스가 종료되는 흐름은 크게 두 가지이다.

- 유저 어플리케이션에서 exit() 함수를 호출
- 종료 시그널 전달받음

## 4.7.1 프로세스 종료 흐름 파악

커널에서 제공하는 do_exit() 함수를 실행하면 프로세스를 종료할 수 있다.<br>
do_exit() 함수에서 커널이 프로세스를 종료하는 코드를 분석하기 전에 do_exit() 함수의 호출되는지 살펴보자.

### exit() 시스템 콜 실행

유저 공간에서 exit 시스템 콜을 발생시켰을 때 프로세스가 종료되는 흐름이다.<br>
보통 유저 프로세스가 정해진 시나리오에 다라 종료해야 할 때 exit() 함수를 호출한다.<br>
시스템 콜이 발생된 후 해당 시스템 콜 핸들러인 sys_group_exit() 함수가 호출된다.<br>
이후 do_exit() 함수를 호출한다.

### 종료 시그널을 전달받았을때

kill 시그널을 받아 프로세스가 소멸하는 흐름이다.<br>
유저 프로세스뿐만 아니라 커널 프로세스도
커널 내부에서 종료 시그널을 받으면 소멸된다.(4.4.2의 실습을 참고해보자)<br>
종료 시그널을 받은 프로세스는 do_exit() 함수를 실행해 소멸된다.<br>

### 왜 프로세스의 종료 흐름을 파악해야 하는가

유저 어플리케이션 프로세스나 커널 프로세스가 예외 상황에서
의도치 않게 종료해서 문제가 발생하는 경우가 있기 때문이다.

## 4.7.2 do_exit() 함수 분석

do_exit() 함수의 이름만 보아도 '종료를 실행한다'라는 동작을 예상할 수 있다.<br>
여기서 '종료를 실행한다'의 주체는 프로세스다.<br>
do_exit() 함수를 분석하면서 프로세스가 종료되는 과정을 살펴보자.

### do_exit() 함수 선언부와 인자 확인

> <https://github.com/raspberrypi/linux/blob/rpi-4.19.y/kernel/exit.c>

```c
void __noreturn do_exit(long code);
```

code라는 인자는 프로세스 종료 코드를 의미한다.<br>
만약 터미널에서 "kill -9 [pid]"라는 명령어를 입력해 프로세스를 종료하면<br>
code 인자로 9가 전달된다.

다음은 TRACE32로 do_exit() 함수를 호출했을 때의 콜 스택 정보이다.

```bash
-000|do_exit(?)
-001|do_group_exit(exit_code=9)
-002|get_signal(?)
-003|do_notify_resume(regs = 0A423EC0, thread_flags = 9)
-004|work_pending(asm)
```

001번째 줄을 보면 exit_code가 9이다.<br>
"kill -9 [PID]" 명령어로 프로세스를 종료하니 exit_code 인자로 9가 전달된 것이다.

do_exit() 함수의 선언부 왼쪽을 보면 \__noreturn 키워드가 있다.<br>
이 지시자는 실행후 자신을 호출한 함수로 되돌아가지 않는다는 뜻이다.<br>
커널 코드를 실행하는 주인공인 프로세스가 종료되니 당연히 이전 함수로 되돌아가지 못한다.<br>
따라서 반환값은 없다.<br>

do_exit() 함수에서 do_task_dead() 함수를 호출해서<br>
schedule() 함수를 실행함으로써 함수 흐름을 마무리하는 이유는 무엇일까?<br>
프로세스는 자신의 프로세스 스택 메모리 공간을 해제할 수 없기 때문이다.<br>
즉, 프로세스를 해제하는 동작인 do_exit() 함수를 스택 메모리 공간에서 실행하기 때문이다.

자신의 프로세스 스택 공간을 해제하는 일을
프로세스 스택 공간에서 실행하는 것은 불가능 하기 때문이다.<br>
따라서 schedule() 함수를 호출해 스케줄링한 후<br>
'다음에 실행되는 프로세스'가 동료되는 프로세스의 스택 메모리 공간을 해제시켜 준다.<br>
이를 위해 do_task_dead(), schedule() 함수를 호출해서 do_exit() 함수 실행을 마무리하는 것이다.

### do_exit() 함수의 동작 방식 확인

do_exit() 함수의 실행 단계는 다음과 같이 정리할 수 있다.

1. init 프로세스가 종료하면 강제 커널 패닉 유발: 보통 부팅 과정에서 발생
2. 이미 프로세스가 do_exit() 함수의 실행으로 프로세스가 종료되는 도중 다시 do_exit() 함수가 호출됐는지 점검
3. 프로세스 리소스(파일 디스크립터, 가상 메모리, 시그널) 등을 해제
4. 부모 프로세스에게 자신이 종료되고 있다고 알림
5. 프로세스의 실행 상태를 task_struct 구조체의 state 필드에 TASK_DEAD로 설정
6. do_task_dead() 함수를 호출해 스케줄링을 실행<br>
do_task_dead() 함수에서 \__schedule() 함수가 호출되어 프로세스 자료구조인 태스크 디스크립터와 스택 메모리 해제

### do_exit() 함수 코드 분석

> <https://github.com/raspberrypi/linux/blob/rpi-4.19.y/kernel/exit.c>

```c
void __noreturn do_exit(long code)
{
    struct task_struct *tsk = current;
    int group_dead;
 
...

    if (unlikely(tsk->flags & PF_EXITING)) {
        pr_alert("Fixing recursive fault but reboot is needed!\n");
        /*
        * We can do this unlocked here. The futex code uses
        * this flag just to verify whether the pi state
        * cleanup has been done or not. In the worst case it
        * loops once more. We pretend that the cleanup was
        * done as there is no way to return. Either the
        * OWNER_DIED bit is set by now or we push the blocked
        * task into the wait for ever nirwana as well.
        */
        tsk->flags |= PF_EXITPIDONE;
        set_current_state(TASK_UNINTERRUPTIBLE);
        schedule();
    }

...

    exit_signals(tsk); /* sets PF_EXITING */

...

    exit_mm();

...

    exit_files(tsk);
    exit_fs(tsk);

...

    exit_notify(tsk, group_dead);

...

    do_task_dead();

...

}
```

```c
    if (unlikely(tsk->flags & PF_EXITING)) {
        pr_alert("Fixing recursive fault but reboot is needed!\n");

        tsk->flags |= PF_EXITPIDONE;
     set_current_state(TASK_UNINTERRUPTIBLE);
      schedule();
    }   
```

태스크 디스크립터를 나타내는 task_struct 구조체의 flags에 PX_EXITING 플래그가 설정됐을 때 코드를 실행한다.<br>
이것은 프로세스가 do_exit() 함수를 실행하는 도중에<br>
다시 do_exit() 함수가 호출됐을 때 예외를 처리하는 코드이다.

task_struct 구조체의 flags에 PF_EXITPIDONE 플래그를 설정한다.<br>
프로세스의 디스크립터 필드인 state에 TASK_UNINTERRUPTIBLE로 상태를 지정한다.<br>
다음 줄에서는 schedule() 함수를 호출해서 휴면상태에 진입한다.

- do_exit() 함수 내에서 exit_signals() 함수가 실행되면
task_struct 구조체의 flags를 PF_EXITING 플래그로 설정한다.

    > <https://github.com/raspberrypi/linux/blob/rpi-4.19.y/kernel/signal.c>

    ```c
    void exit_signals(struct task_struct *tsk)
    {
    ...
        task->flags |= PF_EXITING;
    ...
    }
    ```

    task_struct 구조체의 flags가 PF_EXITING 플래그이면 현재 프로세스가 do_exit() 함수를 실행하는 중
    이라는 뜻이다.

```c
    exit_signals(tsk); /* sets PF_EXITING */
```

프로세스의 task_struct 구조체의 flags 필드를 PF_EXITING으로 바꾼다.<br>
종료할 프로세스가 처리할 시그널이 있으면 retarget_shared_pending() 함수를 실행해
시그널을 대신 처리할 프로세스를 선정한다.

```c
    exit_mm();
```

프로세스의 메모리 디스크립터인 mm_struct 구조체의 리소스를 해제하고
메모리 디스크립터의 사용 카운트를 1만큼 감소한다.

```c
    exit_files(tsk);
    exit_fs(tsk);
```

프로세스가 사용하고 있는 파일 디스크립터 정보를 해제한다.

```c
    exit_notify(tsk, group_dead);
```

부모 프로세스에게 현재 프로세스가 종료 중이라는 사실을 통지한다.

```c
   do_task_dead();
```

이미 do_exit() 함수에서 메모리와 파일 리소스를 해제했고,
부모 프로세스에게 자신이 종료 중이라는 사실도 통보했다.<br>
마지막은 소멸 단계이다.

- 부팅 도중 init 프로세스가 종료되는 경우가 있다. 이때 다음과 같은 커널 로그와 함께 커널 패닉이 발생한다.

    ```bash
    [...][4] Kernel panic - not syncing: Attempted to kill init! exitcode=0x000000b
    [...][4]
    [...][6] CPU2: stopping
    [...][6] CPU: 2 PID: 339 Comm: mmc-cmdpq/0 Tainted: P
    ```

    유저 프로세스의 부모 프로세스가 종료되면 init 프로세스가 부모 프로세스가 된다.<br>
    init 프로세스는 유저 레벨 프로세스를 생성하며 관정하는 역할을 수행한다.<br>
    그런데 init 프로세스가 종료된다면 시스템은 정상적으로 동작할 수 없는 상황이다.<br>
    init 프로세스가 불의의 상황으로 종료되면 강제로 커널 패닉을 발생시킨다.<br>
    보통 리눅스 커널 버전을 업그레이드한 후 root 파일 시스템이나 시스템을 초기화하는 데
    필요한 디바이스 노드를 생성하지 못했을 때 init 프로세스가 종료된다.

## 4.7.3 do_task_dead() 함수 분석

> <https://github.com/raspberrypi/linux/blob/rpi-4.19.y/kernel/sched/core.c>

```c
void __noreturn do_task_dead(void)
{
    /* Causes final put_task_struct in finish_task_switch(): */
    set_special_state(TASK_DEAD);

    /* Tell freezer to ignore us: */
    current->flags |= PF_NOFREEZE;

    __schedule(false);
    BUG();

    /* Avoid "noreturn function does return" - but don't continue if BUG() is a NOP: */
    for (;;)
        cpu_relax();
}
```

set_special_state() 함수를 호출해 프로세스의 상태를 TASK_DEAD 플래그로 바꾼다.<br>
그 다음에는 프로세스 태스크 디스크립터의 flags 필드에 PF_NOFREEZE 플래그를 OR 연산으로 적용한다.

## 4.7.4 do_task_dead() 함수를 호출하고 난 후의 동작

do_task_dead() 함수에서 \__schedule() 함수를 호출하고 나면
커널은 어떻게 프로세스를 소멸시키는지 알아보자.

프로세스는 스스로 자신의 스택 메모리 공간을 해제하지 못하므로
컨텍스트 스위칭을 수행한 후 다음에 실행되는 프로세스에게 자신의 스택 메모리 공간을 해제하고
소멸을 요청한다.

### \__schedule() 함수

> <https://github.com/raspberrypi/linux/blob/rpi-4.19.y/kernel/sched/core.c>

```c
static void __sched notrace __schedule(bool preempt)
{
...
    if (likely(prev != next)) {
    ...
        rq = context_switch(rq, prev, next, &rf);
    } else {
        rq->clock_update_flags &= ~(RQ_ACT_SKIP|RQCF_REQ_SKIP);
        rq_unlock_irq(rq, &rf);
    }
...
}
```

4번째 줄에서 context_switch() 함수를 호출해 컨텍스트 스위칭을 실행한다.

### context_swithing() 함수

> <https://github.com/raspberrypi/linux/blob/rpi-4.19.y/kernel/sched/core.c>

```c
static __always_inline struct rq *
context_switch(struct rq *rq, struct task_struct *prev,
           struct task_struct *next, struct rq_flags *rf)
{
...
    switch_to(prev, next, prev);
    barrier();

    return finish_task_switch(prev);
}
```

8번째 줄에서는 finish_task_switch() 함수를 호출한다.
schedule() 함수를 호출하면 결국 finish_task_switch() 함수가 호출된다.

- \__schedule() 함수와 context_switch() 함수를 실행해 프로세스가 컨텍스트 스위칭이 이뤄지는 과정은 10장 '스케줄링'의 10.9절을 참고하자.

### finish_task_switch() 함수

> <https://github.com/raspberrypi/linux/blob/rpi-4.19.y/kernel/sched/core.c>

```c
static struct rq *finish_task_switch(struct task_struct *prev)
__releases(rq->lock)
{
...
    if (unlikely(prev_state == TASK_DEAD)) {
        if (prev->sched_class->task_dead)
            prev->sched_class->task_dead(prev);
        
        kprobe_flush_task(prev);

        put_task_stack(prev);

        put_task_struct(prev);
    }
```

4번째 줄은 이전에 실행했던 프로세스 상태가 TASK_DEAD일 때 이하 코드를 실행하는 조건문이다.
10번재 줄에서는 put_task_stack() 함수를 호출해서
프로세스 스택 메모리 공간을 해제하고 커널 메모리 공간에 반환한다.

12번째 줄에서는 put_task_stack() 함수를 실행해
프로세스를 표현하는 자료구조인 task_struct가 위치한 메모리를 해제한다.

### finish_trace_switch() 함수 ftrace log

- ftrace 설정 방법은 3.4절을 참고하자.

```c
TaskSchedulerSe-1803  [000] ....  2630.705218: do_exit+0x14/0xbe0 <-do_group_exit+0x50/0xe8
TaskSchedulerSe-1803  [000] ....  2630.705234: <stack trace>
=> do_exit+0x18/0xbe0
=> do_group_exit+0x50/0xe8
=> get_signal+0x160/0x7dc
=> do_signal+0x274/0x468
=> do_work_pending+0xd4/0xec
=> slow_work_pending+0xc/0x20
=> 0x756df704
TaskSchedulerSe-1803  [000] dns.  2630.705438: sched_wakeup: comm=rcu_sched pid=10 prio=120 target_cpu=000
TaskSchedulerSe-1803  [000] dnh.  2630.705466: sched_wakeup: comm=jbd2/mmcblk0p2- pid=77 prio=120 target_cpu=000
TaskSchedulerSe-1803  [000] d...  2630.705479: sched_switch: prev_comm=TaskSchedulerSe prev_pid=1803 prev_prio=120 prev_state=R+ ==> next_comm=rcu_sched next_pid=10 next_prio=120
    rcu_sched-10    [000] d...  2630.705483: finish_task_switch+0x14/0x230 <-__schedule+0x328/0x9b0
    rcu_sched-10    [000] d...  2630.705504: <stack trace>
=> finish_task_switch+0x18/0x230
=> __schedule+0x328/0x9b0
=> schedule+0x50/0xa8
=> rcu_gp_kthread+0xdc/0x9fc
=> kthread+0x140/0x170
=> ret_from_fork+0x14/0x28
```

위 ftrace 메세지는 TaskSchedulerSe-1803 프로세스가 종료되는 과정을 담고 있다.

```c
TaskSchedulerSe-1803  [000] ....  2630.705218: do_exit+0x14/0xbe0 <-do_group_exit+0x50/0xe8
TaskSchedulerSe-1803  [000] ....  2630.705234: <stack trace>
=> do_exit+0x18/0xbe0
=> do_group_exit+0x50/0xe8
=> get_signal+0x160/0x7dc
=> do_signal+0x274/0x468
=> do_work_pending+0xd4/0xec
=> slow_work_pending+0xc/0x20
```

TaskSchedulerSe-1803 프로세스가 종료 시그널을 받고 do_exit() 함수를 호출한다.

```c
TaskSchedulerSe-1803  [000] dns.  2630.705438: sched_wakeup: comm=rcu_sched pid=10 prio=120 target_cpu=000
```

TaskSchedulerSe-1803 프로세스에서 pid가 10인 rcu_sched 프로세스로 스케줄링된다.

```c
    rcu_sched-10    [000] d...  2630.705483: finish_task_switch+0x14/0x230 <-__schedule+0x328/0x9b0
    rcu_sched-10    [000] d...  2630.705504: <stack trace>
=> finish_task_switch+0x18/0x230
=> __schedule+0x328/0x9b0
=> schedule+0x50/0xa8
=> rcu_gp_kthread+0xdc/0x9fc
=> kthread+0x140/0x170
=> ret_from_fork+0x14/0x28
```

rcu_sched 프로세스는 finish_task_switch() 함수에서
TaskSchedulerSe-1803 프로세스의 마지막 리소스를 정리한다.

TaskSchedulerSe-1803 프로세스는 do_exit() 함수에서 태스크 디스크립터의 여러 필드를 해제했다.<br>
그런데 do_exit() 함수를 TaskSchedulerSe-1803 프로세스의 스택 공간에서 실행 중이니
스스로 자신의 스택 공간을 해제할 수 없다.

따라서 <u>스케줄링을 한 후 다음에 실행하는 프로세스인 rcu_sched가 종료되는
TaskSchedulerSe-1803 프로세스의 스택 메모리 공간을 해제하는 것이다.</u>
