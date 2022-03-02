---
title: "4.5 커널 스레드"
excerpt: "커널 공간에서만 실행되는 프로세스인 커널 스레드의 생성 과정을 알아보자"

categories:
    - Linux Kernel Debugging
tags:
    - [linux, kernel, debugging, thread]

toc: true
toc_sticky: true

date: 2022-02-17
last_modifided_at: 2022-02-17
---
## 4.5.1 커널 스레드란?

커널 공간에서만 실행되는 프로세스를 말한다.<br>
대부분 커널 스레드 형태로 동작한다.<br>
커널 스레드는 시스템 메모리나 전원을 제어하는 등<br>
데몬과 비슷한 역할을 하는데<br>
커널 스레드는 유저 영역과 시스템 콜을 받지 않고 동작한다.

커널 스레드의 특징

- 커널 스레드는 커널 공간에서만 실행되며, 유저 공간과 상호작용하지 않는다.
- 커널 스레드는 실행, 휴면 등 모든 동작을 커널에서 직접 제어/관리 한다.
- 대부분의 커널 스레드는 시스템이 부팅할 때 생성되고 시스템이 종료할 때까지 백그라운드로 실행된다.


## 4.5.2 커널 스레드의 종류

### 라즈베리 파이에서 커널 스레드 항목 확인

#### kthreadd 프로세스
```bash
PPID   PID  PGID   SID TTY      TPGID STAT   UID   TIME COMMAND
   0     2     0     0 ?           -1 S        0   0:00 [kthreadd]
```
모든 커널 스레드의 부모 프로세스이다.<br>
스레드 핸들러 함수는 kthreadd()이며, 커널 스레드를 생성하는 역할을 수행한다.

##### 커널 스레드의 핸들러 함수
커널 스레드는 일반 프로세스와 달리 프로세스가 실행하고<br>
휴면 상태에 진입하는 세부 동작을 커널 함수를 사용해 구현해야 한다.<br>
또한 커널 스레드를 생성할 때 호출하는<br>
kthread_create() 함수의 첫번째 인자로 커널 스레드 핸들러 함수를 지정해야 한다.

kthread_create() 선언부

```c
#define kthread_create(threadfn, data, namefmt, arg...) \
	kthread_create_on_node(threadfn, data, NUMA_NO_NODE, namefmt, ##arg)
```
커널 스레드의 세부 동작은 커널 스레드 핸들러 함수에 구현돼 있어서<br>
커널 스레드를 처음 접할 때 먼저 커널 스레드 핸들러 함수를 분석한다.

#### 워커 스레드
```bash
PPID   PID  PGID   SID TTY      TPGID STAT   UID   TIME COMMAND
   2     4     0     0 ?           -1 I<       0   0:02  \_ [kworker/0:0H]
```
워커 스레드는 워크큐에 큐잉된 워크(work)를 실행하는 프로세스이다.<br>
스레드 핸들러 함수는 worker_thread()이며,<br>
process_one_work() 함수를 호출해서 워크를 실행하는 기능을 수행한다.

#### ksoftirpd 프로세스
```bash
PPID   PID  PGID   SID TTY      TPGID STAT   UID   TIME COMMAND
   2     9     0     0 ?           -1 S        0   0:00  \_ [ksoftirqd/0]
```
ksoftirqd 스레드는 이름과 같이 Soft IRQ를 위해 실행하는 프로세스이다.<br>
ksoftirqd 스레드는 smp_boot 형태의 스레드이며, 프로세스 이름의 맨 오른쪽에서<br>
실행 중인 CPU 번호를 볼 수 있다.
예시는 'ksoftirqd/0'이므로 0번 CPU에서만 실행되는 프로세스이고<br>
'ksoftirqd/1'의 경우 1번 CPU에서만 실행되는 프로세스이다.

ksoftirqd 스래드의 핸들러는 run_ksoftirqd() 함수로,<br>
Soft IRQ 서비스를 실행한다.<br>
Soft IRQ 서비스를 처리하는 \_do_softirq() 함수에서 ksoftirq를 깨운다.

#### irq/86-mmc1 스레드
```bash
PPID   PID  PGID   SID TTY      TPGID STAT   UID   TIME COMMAND
   2    78     0     0 ?           -1 S        0   0:00  \_ [irq/86-mmc1]
```
IRQ 스레드라고 하며, 인터럽트 후반부 처리를 위해 쓰이는 프로세스이다.<br>
이름을 정하는 규칙은 다음과 같다.

> ["irq" / "인터럽트 번호"-"인터럽트 이름"]

프로세스의 이름으로 IRQ 스레드가 어떤 기능인지 유추할 수 있다.<br>
예시의 프로세스는 86번 mmc1 인터릅트의 후반부를 처리하는 IRQ 스레드이다.

## 4.5.3 커널 스레드는 어떻게 생성할까?
커널 스레드가 생성되는 과정은 크게 2가지로 나눌 수 있다.

1. kthreadd 프로세스에게 커널 스래드 생성 요청
    - kthread_create()
    - kthread_create_on_node()
2. kthreadd 프로세스가 커널 스래드를 생성
    - kthreadd()
    - create_kthread()

### 1단계 kthreadd 프로세스에게 커널 스레드 생성 요청
유저 프로세스를 생성하려면 유저 어플리케이션에서 fork() 함수를 호출해야 하듯이,<br>
커널 스레드를 생성하려면 kthread_create() 커널 함수를 호출해야 한다.<br>
먼저 kthreadd 프로세스에게 커널 스레드 생성을 요청하는 함수 실행 흐름을 살펴보자.

#### kthread_create() 함수 분석
```c
#define kthread_create(threadfn, data, namefmt, arg...) \
	kthread_create_on_node(threadfn, data, NUMA_NO_NODE, namefmt, ##arg)

struct task_struct *kthread_create_on_node(int (*threadfn)(void *data),
						void *data, int node,
						const char namefmt[],
						...)
```
함수 인자
- int(\*threadfn)(void \*data)<br>
스레드 핸들러 함수 주소를 저장하는 필드이다.<br>
커널 스레드의 세부 동작은 스레드 핸들러 함수에 구현되어 있다.
- void \*data<br>
스레드 핸들러 함수로 전달하는 매개변수다.<br>
주로 주소를 전달하며, 스레드를 식별하는 구조체의 주소를 전달한다.
- int node<br>
- const char namefmt[]<br>
커널 스레드 이름을 저장한다.

커널 스레드를 생성하는 예제 코드를 통해 좀 더 자세하게 알아보자.

> https://github.io/raspberrypi/linux/blob/rpi-4.19/drivers/vhost/vhost.c

```c
long vhost_dev_set_owner(struct vhost_dev *dev)
{
	struct task_struct *worker;
	int err;
...
 	/* No owner, become one	*/
	dev->mm = get_task_mm(current);
	worker = kthread_create(vhost_worker, dev, "vhost-%d", current->pid);
```
8번째 줄을 보면 kthread_create() 함수의 첫번째 인자로<br>
vhost_worker로 스레드 핸들러 함수의 이름을 지정한다.<br>
두번째 인자로 dev 변수를 지정하는데,<br>
첫번째 줄의 함수 인자를 보면 dev가 어떤 구조체인지 알 수 있다.<br>
vhost_dev 구조체의 주소를 두번째 인자로 전달하는 것이다.<br>
이 방식으로 전달하는 인자를 매개변수 혹은 디스크립터라고 부른다.

세번째 인자로 "vhost-%d"를 전달한다. 이는 커널 스레드의 이름을 나타낸다.

##### 스레드 핸들러 함수로 전달되는 매개변수

> https://github.io/raspberrypi/linux/blob/rpi-4.19.y/drivers/vhost/vhost.c

```c
static int vhost_worker(void *data)
{
	struct vhost_dev *dev = data;
	struct vhost_work *work, *work_next;
	struct llist_node *node;
```
vhost_dev_set_owner() 함수에서 kthread_create() 함수를 호출할 때<br>
두번째 인자로 vhost_dev 구조체인 dev를 지정했다.<br>
이 주소가 스레드 핸들러인 vhost_worker() 함수의 인자로 전달되는 것이다.

이처럼 vhost_worker() 스레드 핸들러 함수의 매개변수로 인자를 전달한다.<br>
첫번째 줄에서 보이는 void 타입의 data 포인터를<br>
세번째 줄과 같이 vhost_dev 구조체로 형변환(casting)한다.<br>
커널에서는 이 같은 방식으로 스레드를 관리하는 구조체를 매개변수로 전달한다.

#### kthread_create() 함수 구현부

> https://github.io/raspberrypi/linux/blob/rpi-4.19.y/include/linux/kthread.h

```c
#define kthread_create(threadfn, data, namefmt, arg...) \
 kthread_create_on_node(threadfn, data, NUMA_NO_NODE, namefmt, ##arg)
```
kthread_create() 함수는 선언부와 같이 매크로 타입이다.<br>
따라서 커널 컴파일 과정에서 전처리기는 kthread_create() 함수를<br>
kthread_create_on_node() 함수로 바꾼다.<br>
즉, 커널이다 드라이버 코드에서 kthread_create() 함수를 호출하면<br>
실제로 동작하는 코드는 kthread_create_on_node() 함수인 것이다.

#### kthread_create_on_node() 함수 분석
kthread_create() 매크로 함수의 실체인 kthread_create_on_node() 함수의 구현부를 보자.

> https://github.com/raspberrypi/linux/blob/rpi-4.19.y/kernel/kthread.c

```c
struct task_struct *kthread_create_on_node(int (*threadfn)(void *data),
				void *data, int node, const char namefmt[]
				...)
{
	struct task_struct *task;
	va_list args;

	va_start(args, namefmt);
	task = __kthread_create_on_node(threadfn, data, node, namefmt, args);
	va_end(args);

	return task;
}
```
함수 구현부를 봐도 특별한 일은 하지 않는다.<br>
다만 가변 인자를 통해 \__kthread_create_on_node() 함수를 호출할 뿐이다.<br>
결국 \__kthread_create_on_node() 함수에서 커널 스레드 생성을 요청한다.

> https://github.com/raspberrypi/linux/blob/rpi-4.19.y/kernel/kthread.c

```c
struct task_struct *__kthread_create_on_node(int (*threadfn)void( *data
                 void *data, int node,
                 const char namefmt[],
                 va_list args)
{
    DECLARE_COMPLETION_ONSTACK(done);
    struct task_struct *task;
    struct kthread_create_info *create = kmalloc(sizeof(*create),
                               GFP_KERNEL);

    if (!create)
        return ERR_PTR(-ENOMEM);
    create->threadfn = threadfn;
    create->data = data;
    spin_lock(&kthread_create_lock);
    list_add_tail(&create->list, &kthread_create_list);
    spin_unlock(&thread_create_lock);

    wake_up_process(kthreadd_task);
```

```c
    struct kthread_create_info *create = kmalloc(sizeof(*create), GFP_KERNEL);
```
kmalloc() 함수를 호출해 kthread_create_info 구조체 크기만큼 동적 메모리를 할당받는다.

```c
    create->threadfn = threadfn;
    create->data = data;
    create->node = node;
```
생성할 커널 스레드 핸들러 함수와 매개변수 및 노드를<br>
kthread_create_info 구조체의 필드에 저장한다.
```c
    list_add_tail(&create->list, &kthread_create_list);
```
커널 스레드 생성 요청을 관리하는 kthread_create_list 연결 리스트에 &create->list를 추가한다.<br>
kthreadd 프로세스는 kthread_create_list 연결 리스트를 확인해<br>
커널 스레드 생성 요청이 있었는지 확인한다.
```c
    wake_up_process(kthreadd_task);
```
kthreadd 프로세스의 태스크 디스크립터인 kthreadd_task를 인자로 삼아<br>
wake_up_process() 함수를 호출해 kthreadd 프로세스를 깨운다.

### 2단계: kthreadd 프로세스가 커널 스레드를 생성
kthreadd 프로세스를 깨우면 kthreadd 프로세스의 스레드 핸들러 kthreadd() 함수가 실행을 시작한다.<br>
1단계에서 커널 스레드 생성을 kthreadd 프로세스에게 요청했다.<br>
그리고 kthreadd 프로세스를 깨웠다.<br>
kthreadd 프로세스를 깨우면<br>
kthreadd 프로세스의 스레드 핸들러인 kthreadd() 함수가 호출되어 프로세스르 생성한다.<br>
#### kthreadd() 함수 분석
```c
int kthreadd(void *unused)
{
    struct task_struct *tsk = current;
    /* Setup a clean context for our children to inherit.  */
}
```
