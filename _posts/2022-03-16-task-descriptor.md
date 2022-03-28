---
title: "4.8 태스크 디스크립터"
excerpt: "프로세스를 표현하고 관리하는 프로세스 자료구조"

categories:
    - Linux Kernel Debugging
tags:
    - [linux, kernel, debugging, process, task, descriptor]

toc: true
toc_sticky: true

date: 2022-03-16
last_modified_at: 2022-03-16
---

프로세스의 속성 정보를 표현하는 가장 중요한 자료구조는 태스크 디스크립터를 나타내는
task_struct 구조체이다.

- 임베디드 시스템에서 태스크 혹은 프로세스 정보를 표현하는 자료구조를 **TCB(Task Control Block)**이라고 한다.<br>
    리눅스 커널에서 프로세스 정보를 표현하는 자료구조는 **태스크 디스크립터**이다.

다음 코드 블록은 task_struct 구조체에 접근해서 프로세스 정보를 출력하는 예시이다.

```bash
S   UID   PID  PPID  C PRI  NI   RSS    SZ WCHAN  TTY          TIME CMD
S     0     1     0  0  80   0  8788  8465 -      ?        00:00:06 systemd
S     0     2     0  0  80   0     0     0 -      ?        00:00:00 kthreadd
I     0     3     2  0  60 -20     0     0 -      ?        00:00:00 rcu_gp
I     0     4     2  0  60 -20     0     0 -      ?        00:00:00 rcu_par_gp
I     0     8     2  0  60 -20     0     0 -      ?        00:00:00 mm_percpu_wq
S     0     9     2  0  80   0     0     0 -      ?        00:00:00 rcu_tasks_rude_
S     0    10     2  0  80   0     0     0 -      ?        00:00:00 rcu_tasks_trace
S     0    11     2  0  80   0     0     0 -      ?        00:00:00 ksoftirqd/0
I     0    12     2  0  80   0     0     0 -      ?        00:00:01 rcu_sched
```

맨 오른쪽에 systemd, kthreadd 순서로 프로세스 이름을 볼 수 있고,
PID는 1, 2, 3번으로 확인할 수 있다.
이 프로세스 정보는 태스크 디스크립터 필드에 저장된 값을 읽어서 표현하는 것이다.

리눅스 시스템에서 구동 중인 프로세스 모록은 프로세스를 관리하는 **태스크 디스크립터의 연결 리스트(init_task.tasks)**에 접근해서 등록된 프로세스를 출력한다.

## 4.8.1 프로세스를 식별하는 필드

프로세스를 식별하는 필드는 다음과 같다.

```c
char comm[TASK_COMM_LEN];
```

comm은 TASK_COMM_LEN 크기의 배열이며, 프로세스 이름을 저장한다.

```bash
S   UID   PID  PPID  C PRI  NI   RSS    SZ WCHAN  TTY          TIME CMD
S     0     1     0  0  80   0  8788  8465 -      ?        00:00:06 systemd
S     0     2     0  0  80   0     0     0 -      ?        00:00:00 kthreadd
I     0     3     2  0  60 -20     0     0 -      ?        00:00:00 rcu_gp
```

출력 결과에서 맨 오른쪽 부분에서 systemd, kthreadd, rcu_gp가 보인다.<br>
**프로세스 이름들은 태스크 디스크립터를 나타내는 task_struct 구조체의 comm 필드에 접근해서 출력하는 것이다.**<br>
프로세스 이름은 set_task_comm() 함수를 호출해서 지정할 수 있다.

- 디버깅 코드

    > <https://github.com/raspberrypi/linux/blob/rpi-4.19.y/kernel/sched/core.c>

    ```c
    static void ttwu_do_wakeup(struct rq *rq, struct task_struct *p, int wake_flags, struct rq_flags *rf)
    {
        check_preempt_curr(rq, p, wake_flags);
        p->state = TASK_RUNNING;
        trace_sched_wakeup(p);
    
    +   if (!strcmp(p->comm, "kthreadd")) {
    +       printk("[+][%s] wakeup kthreadd process \n", current->comm);
    +       dump_stack();
    +   }

    #ifdef CONFIG_SMP
        if (p->sched_class->task_woken) {
    ```

    ttwu_do_wakeup() 함수의 인자인
    task_struct *p가 깨우려는 태스크 디스크립터이다.<br>
    프로세스의 이름은 task_struct 구조체의 comm 필드에 저장돼 있으며,
    프로세스의 이름이 "kthreadd"인 경우 dump_stack() 함수를 호출해 함수 호출 흐름을 커널 로그로 출력한다.

    위 코드는**kthreadd 프로세스를 깨울 때
    함수 호출 흐름을 커널 로그로 출력하기 위한 코드이다.**

```c
pid_t pid;
```

pid는 Process ID의 약자로 프로세스마다 부여하는 정수형 값이다.<br>
pid 상수는 프로세스를 생성할 때마다 증가하므로
pid 값이 크기로 프로세스가 언제 생성됐는지 추정할 수 있다.

```c
pid_t tgid;
```

pid와 같은 타입의 필드로 스레드 그룹 아이디를 표현하는 정수형 값이다.<br>
해당 프로세스가 스레드 리더인 경우는 tgid와 pid가 같고,
자식 스레드인 경우는 tgid와 pid가 다르다.

## 4.8.2 프로세스 상태 저장

태스크 디스크립터에는 프로세스 상태를 관리하는 두 가지 필드가 있다.

- state: 프로세스 실행 상태
- flags: 프로세스 세부 동작 상태와 속성 정보

### state 필드

```c
volatile long state;
```

- volatile 변수는 최적화에서 제외되어 항상 메모리에 접근하도록 한다.

> <https://github.com/raspberrypi/linux/blob/rpi-4.19.y/include/linux/sched.h>

```c
#define TASK_RUNNING            0x0000
#define TASK_INTERRUPTIBLE      0x0001
#define TASK_UNINTERRUPTIBLE    0x0002
```

위 매크로에서 정의한 프로세스의 상태는 다음과 같다.

- TASK_RUNNING: CPU에서 실행 중이거나 런큐에서 대기 상태에 있음
- TASK_INTERRUPTIBLE: 휴면 상태
- TASK_UNINTERRUPTIBLE: 특정 조건에서 깨어나기 위해 휴면 상태로 진입한 상태

리눅스 시스템에서 확인되는 프로세스들은 대부분 TASK_INTERRUPTIBLE 상태이다.<br>
TASK_RUNNING 혹은 TASK_UNUNTERRUPTIBLE 상태인 프로세스가
비정상적으로 많으면 시스템에 문제(데드락, 특정 프로세스 스톨)가 있는 경우가 많다.

### flags 필드

```c
unsigned int flags;
```

프로세스 종료와 프로세스 세부 실행 상태를 저장하는 필드이다.<br>
flag 필드는 PF_*로 시작하는 매크로 필드를 OR 연산한 결과를 저장한다.

> <https://github.com/raspberrypi/linux/blob/rpi-4.19.y/include/linux/sched.h>

```c
#define PF_IDLE			    0x00000002	/* I am an IDLE thread */
#define PF_EXITING		    0x00000004	/* Getting shut down */
#define PF_EXITPIDONE		0x00000008	/* PI exit done on shut down */
#define PF_WQ_WORKER		0x00000020	/* I'm a workqueue worker */
#define PF_KTHREAD		    0x00200000	/* I am a kernel thread */
```

- PF_IDLE: 아이들 프로세스
- PF_EXITING: 프로세스가 종료 중인 상태
- PF_EXITPIDONE: 프로세스가 종료를 마무리한 상태
- PF_WQ_WORKER: 프로세스가 워커 스레드인 경우
- PF_KTHREAD: 프로세스가 커널 스레드인 경우

flags 필드에 저장된 값으로 프로세스의 세부 실행 상태를 알 수 있다.<br>
커널은 flags에 저장된 프로세스의 세부 동작 상태를 읽어서 프로세스를 제어한다.

#### flags 필드로 프로세스의 실행 흐름을 제어

> <https://elixir.bootlin.com/linux/v3.7/source/drivers/staging/android/lowmemorykiller.c>

```c
static int lowmem_shrink(struct shrinker *s, struct shrink_control *sc)
{
	struct task_struct *tsk;
	struct task_struct *selected = NULL;
	int rem = 0;
...
    for_each_process(tsk) {
		struct task_struct *p;
		int oom_score_adj;

		if (tsk->flags & PF_KTHREAD)
			continue;

		p = find_lock_task_mm(tsk);
		if (!p)
			continue;

		if (test_tsk_thread_flag(p, TIF_MEMDIE) &&
		    time_before_eq(jiffies, lowmem_deathpending_timeout)) {
			task_unlock(p);
			rcu_read_unlock();
			return 0;
		}
		oom_score_adj = p->signal->oom_score_adj;
		if (oom_score_adj < min_score_adj) {
			task_unlock(p);
			continue;
		}
...
        lowmem_print(2, "select %d (%s), adj %d, size %d, to kill\n",
		            p->pid, p->comm, oom_score_adj, tasksize);
	}
```

위 코드는 안드로이드에서 메모리 회수를 위해 특정 프로세스를 종료하는 lowmem_shrink() 함수의 일부분이다.

```c
    if (tsk->flags & PF_KTHREAD)
        continue;
```

이 코드는 태스크 디스크립터의 flags 필드와
PF_KTHREAD 매크로를 대상으로 AND 비트 연산을 수행한다.<br>
이 조건을 만족하면 continue를 실행해 아래 부분의 코드를 실행하지 않는다.<br>

이 코드의 의도는
**프로세스 타입이 커널 스레드이면 프로세스를 종료하지 않겠다는 것이다.**

이 같은 방식으로 프로세스 실행 정보를 읽어서 예외 처리를 수행한다.

### exit_state 필드

```c
int exit_code;
```

> <https://github.com/raspberrypi/linux/blob/rpi-4.19.y/kernel/exit.c>

```c
void __noreturn do_exit(long code)
{
...
    tsk->exit_code = code;
```

do_exit() 함수를 호출할 때 인자로
다음과 같은 시그널이나 프로세스를 종료하는 옵션을 지정할 수 있다.

> <https://github.com/raspberrypi/linux/blob/rpi-4.19.y/arch/arm/mm/fault.c>

```c
static void
__do_kernel_fault(struct mm_struct *mm, unsigned long addr, unsigned int fsr,
            struct pt_regs *regs)
{
...
    do_exit(SIGKILL);
```

5번째 줄을 보면 SIGKILL 인자와 함께 do_exit() 함수를 호출한다.

## 4.8.3 프로세스 간의 관계

커널에서 프로세스는 다양한 방식으로 서로 연결되어 있다.<br>
태스크 디스크립터에서 프로세스 사이의 관계를 나타내는 다음과 같은 필드에 대해 알아보자.

- struct task_struct *real_parent
- struct task_struct *parent
- struct list_head children
- struct list_head sibling

유저 공간에서 생성한 프로세스의 부모는 대부분 init이고
커널 공간에서 생성한 커널 스레드(프로세스)의 부모는 kthreadd 프로세스이다.<br>
태스크 디스크립터에서는 프로세스의 부모와 자식 관계를 real_parent와 parent 필드로 알 수 있다.

- struct task_struct *real_parent
    자신을 생성한 부모 프로세스의 태스크 디스크립터 주소를 저장한다.
- struct task_struct *parent
    부모 프로세스의 태스크 디스크립터 주소를 담고 있다.

### \*real_parent와 *parent 필드의 차이점

일반적인 상황에서 자신을 생성한 부모 프로세스가 종료되지 않고 실행 중이면
real_parent와 parent가 같다.<br>
그런데 부모 프로세스가 소멸될 수 있다.<br>
프로세스 계층 구조에서 지정한 부모 프로세스가 없을 경우 init 프로세스를 부모 프로세스로 변경한다.
이 조건에서는 \*real_parent와 *parent 필드가 다르다.<br>

부모 프로세스가 종료되면 다음과 같은 순서로 함수를 호출한다.

- sys_exit_group
- do_exit_group
- do_exit
- exit_notify
- forget_original_parent
- find_new_reaper

함수 이름과 같이 forget_original_parent() 함수와
find_new_reaper() 함수에서 새로운 부모 프로세스를 지정한다.

### 프로세스 간의 관계를 저장하는 children과 sibling 필드

- struct list_head children
    부모 프로세스가 자식 프로세스를 생성할 때 children 연결 리스트에 자식 프로세스를 등록
- struct list_head sibling
    같은 부모 프로세스로 생성된 프로세스의 연결 리스트 주소를 저장한다.<br>
    필드명에서 알 수 있듯이 형제 관계의 프로세스들이 등록된 연결 리스트이다.

```bash
root@raspberrypi:/home/pi# ps axjf
 PPID   PID  PGID   SID TTY      TPGID STAT   UID   TIME COMMAND
    0     2     0     0 ?           -1 S        0   0:00 [kthreadd]
    2     3     0     0 ?           -1 I<       0   0:00  \_ [rcu_gp]
    2     4     0     0 ?           -1 I<       0   0:00  \_ [rcu_par_gp]
    2     8     0     0 ?           -1 I<       0   0:00  \_ [mm_percpu_wq]
```

출력 결과, rcu_gp, rcu_par_gp, mm_percpu_wq 프로세스들의
부모 프로세스는 kthreadd임을 알 수 있다.<br>
각 프로세스의 PPID(Parent Process ID) 항목을 보면 모두 2인데,
커널 스레드를 생성하는 kthreadd 프로세스의 pid가 2이다.

- 리눅스를 탑재한 대부분의 시스템에서 kthreadd 프로세스의 PID는 2이다.

태스크 디스크립터 관점에서 "kthreadd" 프로세스 태스크 디스크립터의
children 필드는 연결리스트이다.<br>
연결 리스트 헤드에 등록된 자식 프로세스의 task_struct 구조체의 sibling 필드 주소를 저장한다.

"kthreadd" 프로세스의 자식 프로세스인 "rcu_gp" 입장에서 "rcu_par_gp"와 "mm_percpu_wq" 프로세스는
자신의 sibling 연결 리스트로 이어져 있다.<br>
같은 부모 프로세스에서 생성된 프로세스이기 때문이다.

## 4.8.4 프로세스 연결 리스트

task_struct 구조체의 tasks 필드는
list_head 구조체로서 연결 리스트 타입이다.<br>
커널에서 구동 중인 모든 프로세스는 tasks 연결 리스트에 등록돼 있다.<br>

**그렇다면 프로세스의 태스크 디스크립터 필드 중 연결 리스트 타입인 tasks 필드는<br>
언제 init 프로세스의 태스크 디스크립터 필드 중 연결 리스트 타입인 tasks 필드에 등록되는 걸까?**

프로세스는 처음 생성될 때 init_task 전역번수 필드인 tasks 연결 리스트에 등록된다.<br>
프로세스를 생성할 때 호출되는 copy_process() 함수를 보면서 처리 과정을 살펴보자.

> <https://github.com/raspberrypi/linux/blob/rpi-4.19.y/kernel/fork.c>

```c
static __latent_entropy struct task_struct *copy_process(
					unsigned long clone_flags,
					unsigned long stack_start,
					unsigned long stack_size,
					int __user *child_tidptr,
					struct pid *pid,
					int trace,
					unsigned long tls,
					int node)
{
    struct task_struct *p;
    p = dup_task_struct(current, node);
...
    list_add_tail_rcu(&p->tasks, &init_task.tasks);
```

12번째 줄을 실행하면 커널로부터 태스크 디스크립터를 할당받는다.<br>
다음 줄에서는
**init_task.tasks 연결 리스트의 마지막 노드에
현재 프로세스의 taks_struct 구조체의 tasks 주소를 등록한다.**

## 4.8.5 프로세스 실행 시각 정보

태스크 디스크립터에는 프로세스의 실행 시각 정보를 알 수 있는 다음과 같은 필드가 있다.

- u64 utime
- u64 stime
- struct sched_info.last_arrival

### utime 필드

```c
u64 utime
```

유저 모드에서 프로세스가 실행한 시각을 나타낸다.<br>
이 필드는 account_user_time() 함수의 6번째 줄에서 바뀐다.

> <https://github.com/raspberrypi/linux/blob/rpi-4.19.y/kernel/sched/cputime.c>

```c
void account_user_time(struct task_struct *p, u64 cputime)
{
	int index;

	/* Add user time to process. */
	p->utime += cputime;
```

### stime 필드

```c
u64 stime
```

> <https://github.com/raspberrypi/linux/blob/rpi-4.19.y/kernel/sched/cputime.c>

```c
void account_system_index_time(struct task_struct *p,
			       u64 cputime, enum cpu_usage_stat index)
{
	/* Add system time to process. */
	p->stime += cputime;
```

### sched_info.last_arrival 필드

```c
struct sched_info sched_info.last_arrival
```

sched_info 필드는 프로세스 스케줄링 정보를 저장한다.<br>
이 가운데 last_arrival은 프로세스가 마지막 CPU에서 실행된 시간을 나타낸다.

> <https://github.com/raspberrypi/linux/blob/rpi-4.19.y/kernel/sched/stats.h>

```c
static void sched_info_arrive(struct rq *rq, struct task_struct *t)
{
	unsigned long long now = rq_clock(rq), delta = 0;

	if (t->sched_info.last_queued)
		delta = now - t->sched_info.last_queued;
	sched_info_reset_dequeued(t);
	t->sched_info.run_delay += delta;
	t->sched_info.last_arrival = now;
	t->sched_info.pcount++;

	rq_sched_info_arrive(rq, delta);
}
```

9번째 줄을 보면 현재 시각 정보인 now를 't->sched_info.last_arrival'에 저장한다.

**sched_info_arrive() 함수는 언제 호출되는 것일까?**

context_switch() 함수 내에서 컨텍스트 스위칭을 수행하기 직전에
prepare_task_swithc() 함수를 호출한다.<br>
이 prepare_task_switch() 함수가 호출하는 함수를 따라가다 보면
sched_info_arrive() 함수가 호출된다.

> <https://github.com/raspberrypi/linux/blob/rpi-4.19.y/kernel/sched/core.c>

```c
static __always_inline struct rq *
context_switch(struct rq *rq, struct task_struct *prev,
	       struct task_struct *next, struct rq_flags *rf)
{
	struct mm_struct *mm, *oldmm;

	prepare_task_switch(rq, prev, next);
```

context_switch() 함수를 시작으로 다음 순서로 함수를 호출해 sched_info_arrive() 함수가 실행된다.

- context_switch()
- prepare_task_switch()
- sched_info_switch()
- __sched_info_switch()
- sched_info_arrive()

context_switch() 함수 내에서 컨텍스트 스위칭을 수행하기 직전에 위와 같은 흐름으로
sched_info_arrive() 함수가 호출된다.<br>
커널은 sched_info_arrive() 함수에서 프로세스의 실행 시간을 업데이트한다.
