---
title: "4.6 커널 내부 프로세스의 생성 과정"
excerpt: "\_do_fork() 함수를 분석하면서 커널에서 프로세스를 생성하는 과정을 살펴보자"

categories:
    - Linux Kernel Debugging
tags:
    - [linux, kernel, debugging, process, _do_fork]

toc: true
toc_sticky: true

date: 2022-03-07
last_modified_at: 2022-03-07
---
## 4.6.1 \_do_fork() 함수
- 1단계: 프로세스 생성<br>
copy_process() 함수를 호출해서 프로세스를 생성한다.<br>

copy_process() 함수는 부모 프로세스의 리소스를 자식 프로세스에게 복제한다.
- 2단계: 생성한 프로세스의 실행 요청<br>
copy_process() 함수를 호출해 프로세스를 만든 후 wake_up_new_task() 함수를 호출해<br>
스케줄러에게 프로세스 실행 요청을 한다.

> https://github.com/raspberry/linux/blob/rpi-4.19.y/kernel/fork.c

```c
long _do_fork(unsigned long clone_flags,
	      unsigned long stack_start,
	      unsigned long stack_size,
	      int __user *parent_tidptr,
	      int __user *child_tidptr,
	      unsigned long tls)
{
	struct completion vfork;
	struct pid *pid;
	struct task_struct *p;
	int trace = 0;
	long nr;

	/*
	 * Determine whether and which event to report to ptracer.  When
	 * called from kernel_thread or CLONE_UNTRACED is explicitly
	 * requested, no event is reported; otherwise, report if the event
	 * for the type of forking is enabled.
	 */
	if (!(clone_flags & CLONE_UNTRACED)) {
		if (clone_flags & CLONE_VFORK)
			trace = PTRACE_EVENT_VFORK;
		else if ((clone_flags & CSIGNAL) != SIGCHLD)
			trace = PTRACE_EVENT_CLONE;
		else
			trace = PTRACE_EVENT_FORK;

		if (likely(!ptrace_event_enabled(current, trace)))
			trace = 0;
	}

	p = copy_process(clone_flags, stack_start, stack_size,
			 child_tidptr, NULL, trace, tls, NUMA_NO_NODE);
	add_latent_entropy();

	if (IS_ERR(p))
		return PTR_ERR(p);

	/*
	 * Do this prior waking up the new thread - the thread pointer
	 * might get invalid after that point, if the thread exits quickly.
	 */
	trace_sched_process_fork(current, p);

	pid = get_task_pid(p, PIDTYPE_PID);
	nr = pid_vnr(pid);

	if (clone_flags & CLONE_PARENT_SETTID)
		put_user(nr, parent_tidptr);

	if (clone_flags & CLONE_VFORK) {
		p->vfork_done = &vfork;
		init_completion(&vfork);
		get_task_struct(p);
	}

	wake_up_new_task(p);

	/* forking complete and child started to run, tell ptracer */
	if (unlikely(trace))
		ptrace_event_pid(trace, pid);

	if (clone_flags & CLONE_VFORK) {
		if (!wait_for_vfork_done(p, &vfork))
			ptrace_event_pid(PTRACE_EVENT_VFORK_DONE, pid);
	}

	put_pid(pid);
	return nr;
}
```
32번째 줄을 보자.

```c
	p = copy_process(clone_flags, stack_start, stack_size,
			 child_tidptr, NULL, trace, tls, NUMA_NO_NODE);
```
copy_process() 함수를 호출해 부모 프로세스의 메모리 및 시스템 정보를 자식 프로세스에게 복사한다.<br>
이어지는 조건문을 보자.
```c
	if (IS_ERR(p))
		return PTR_ERR(p);
```
p라는 포인터형 변수의 오류를 검사한다.<br>
만약 태스크 디스크립터의 주소를 담고 있는 p 포인터 변수에 오류가 있으면<br>
오류코드를 반환하면서 함수의 시행을 종료한다.
```c
	trace_sched_process_fork(current, p);

	pid = get_task_pid(p, PIDTYPE_PID);
	nr = pid_vnr(pid);
```
43번째 줄은 ftrace 이벤트 중 sched_process_fork를 활성화했을 때 동작한다.<br>
그리고 pid를 계산해서 nr 지역변수에 저장한다.<br>
정수형 타입인 nr변수는 프로세스의 pid이며 \_do_fork() 함수가 실행을 종료한 후 반환한다.<br>
```c
	wake_up_new_task(p);
```
생성한 프로세스를 깨운다.
```c
	return nr;
```
프로세스의 PID를 담고 있는 정수형 타입의 nr 지역변수를 반환한다.<br>
프로세스의 PID를 반환하는 동작이다.<br>


\_do_fork() 함수 작동방식
- copy_process() 함수를 호출해 프로세스를 생성
- wake_up_new_task() 함수를 호출해 생성한 프로세스를 깨움
- 생성한 프로세스 PID를 반환

## 4.6.2 copy_process() 함수
프로세스를 생성하는 핵심 동작은 copy_process() 함수에서 수행한다.<br>
대부분 부모 프로세스에 있는 리소스를 복사하는 동작이다.

> https://github.com/raspberry/linux/blob/rpi-4.19.y/kernel/fork.c

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
	int retval;
	struct task_struct *p;
	struct multiprocess_signals delayed;

...

	retval = -ENOMEM;
	p = dup_task_struct(current, node);
	if (!p)
		goto fork_out;

...

	/* Perform scheduler related setup. Assign this task to a CPU. */
	retval = sched_fork(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_policy;

...

	retval = copy_files(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_semundo;
	retval = copy_fs(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_files;
	retval = copy_sighand(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_fs;

...
```
먼저 18번째 줄을 보자.
```c
	p = dup_task_struct(current, node);
	if (!p)
		goto fork_out;
```
dup_task_struct() 함수는 생성할 프로세스의 태스트 디스크립터인<br>
task_struct 구조체와 프로세스가 실행될 스택 공간을 할당한다.<br>
dup_task_struct() 함수를 호출해 태스크 디스크립터를 p에 저장한다.
```c
	retval = sched_fork(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_policy;
```
태스크 디스크립터를 나타내는 task_struct 구조체에서 스케줄링 관련 정보를 초기화한다.
```c
	retval = copy_files(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_semundo;
	retval = copy_fs(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_files;
```
파일 디스크립터 관련 내용(파일 디스크립터, 파일 디스크립터 테이블)을 초기화한다.<br>
부모 file_struct 구조체의 내용을 자식 프로세스에게 복사한다.<br>
만약 프로세스 생성 플래그 중 CLONE_FILES로 프로세스를 생성했을 경우 참조 카운트만 증가한다.<br>
```c
	retval = copy_sighand(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_fs;
```
프로세스가 등록한 시널 핸들러 정보인 sighand_struct 구조체를 생성하고 복사한다.

copy_process() 함수를 실행해 부모 프로세스의 리소스를<br>
새로 생성하는 프로세스의 task_struct 구조체에 복제하는 과정이었다.

## 4.6.3 wake_up_new_task() 함수
프로세스 생성의 마지막 단계로 생성한 프로세스를 깨운다.<br>
이 동작은 wake_up_new_task() 함수가 수행한다.

- 프로세스의 상태를 TASK_RUNNING으로 변경
- 현재 실행 중인 CPU 번호를 thread_info 구조체의 cpu 필드에 저장
- 런큐에 프로세스를 큐잉

> https://github.com/raspberrypi/linux/blob/rpi-4.19.y/kernel/sched/core.c

```c
void wake_up_new_task(struct task_struct *p)
{
	struct rq_flags rf;
	struct rq *rq;

	raw_spin_lock_irqsave(&p->pi_lock, rf.flags);
	p->state = TASK_RUNNING;
#ifdef CONFIG_SMP
	/*
	 * Fork balancing, do it here and not earlier because:
	 *  - cpus_allowed can change in the fork path
	 *  - any previously selected CPU might disappear through hotplug
	 *
	 * Use __set_task_cpu() to avoid calling sched_class::migrate_task_rq,
	 * as we're not fully set-up yet.
	 */
	p->recent_used_cpu = task_cpu(p);
	__set_task_cpu(p, select_task_rq(p, task_cpu(p), SD_BALANCE_FORK, 0));
#endif
	rq = __task_rq_lock(p, &rf);
	update_rq_clock(rq);
	post_init_entity_util_avg(&p->se);

	activate_task(rq, p, ENQUEUE_NOCLOCK);
```
* wake_up_new_task() 함수의 인자인 task_struct \*p는 새롭게 생성된 프로세스의 태스크 디스크립터임

```c
	p->state = TASK_RUNNING;
```
프로세스 상태를 TASK_RUNNING으로 바꾼다.
```c
	__set_task_cpu(p, select_task_rq(p, task_cpu(p), SD_BALANCE_FORK, 0, 1));
```
\__set_task_cpu() 함수를 호출해 프로세스의 thread_info 구조체의 cpu 필드에 현재 실행 중인 CPU 번호를 저장한다.
```c
	rq = __task_rq_lock(p, &rf);
	update_rq_clock(rq);
	post_init_entity_util_avg(&p->se);

	activate_task(rq, p, ENQUEUE_NOCLOCK);
```
런큐 주소를 읽은 다음, active_task() 함수를 호출해 런큐에서 새롭게 생성한 프로세스를 삽입한다.<br>
