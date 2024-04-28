首先git checkout ch3

编程作业是完成 sys_task_info

搜索关键字 sys_task_info

发现 process.rs中，有这样一条注释  YOUR JOB: Finish sys_task_info to pass testcases

那么通过测试的关键就是完成这个代码

先跑一下测试

cd ci-user
make test CHAPTER=3

发现两个[FAIL]
[PASS] found <get_time OK25670! (\d+)>
[PASS] found <Test sleep OK25670!>
[PASS] found <current time_msec = (\d+)>
[PASS] found <time_msec = (\d+) after sleeping (\d+) ticks, delta = (\d+)ms!>
[PASS] found <Test sleep1 passed25670!>
[FAIL] not found <string from task info test>
[FAIL] not found <Test task info OK25670!>

再次执行
[PASS] found <get_time OK970125670! (\d+)>
[PASS] found <Test sleep OK970125670!>
[PASS] found <current time_msec = (\d+)>
[PASS] found <time_msec = (\d+) after sleeping (\d+) ticks, delta = (\d+)ms!>
[PASS] found <Test sleep1 passed970125670!>
[FAIL] not found <string from task info test>
[FAIL] not found <Test task info OK970125670!>

发现出现了一个数字变化，暂时不清楚它的运行机制
25670前面多个了几个数字，9701这个是哪来的？家人们谁懂啊

先看看第一个没找到string from task info test

在网上滚动输出内容的时候，发现了一段代码
Panicked at src/bin/ch3_taskinfo.rs:19, assertion `left == right` failed
  left: 0
 right: -1

把返回值-1改成0
pub fn sys_task_info(_ti: *mut TaskInfo) -> isize {
    trace!("kernel: sys_task_info");
    -1
}

Panicked at src/bin/ch3_taskinfo.rs:21, assertion failed: 3 <= info.syscall_times[SYSCALL_GETTIMEOFDAY]

assert!(3 <= info.syscall_times[SYSCALL_GETTIMEOFDAY]);

任务使用的系统调用及调用次数

也就是说需要在系统调用时更新调用次数，SYSCALL_GETTIMEOFDAY的调用次数大于3次
看这段代码，调用了4次get_time，应该是在get_time的时候调用了SYSCALL_GETTIMEOFDAY
let t1 = get_time() as usize;
let mut info = TaskInfo::new();
get_time();
sleep(500);
let t2 = get_time() as usize;
// 注意本次 task info 调用也计入
assert_eq!(0, task_info(&mut info));
let t3 = get_time() as usize;

但是在get_time中为找到有关SYSCALL_GETTIMEOFDAY的内容

继续搜索SYSCALL_GETTIMEOFDAY相关内容

pub const SYSCALL_GETTIMEOFDAY: usize = 169;

发现这个值是169

/// gettime syscall
const SYSCALL_GET_TIME: usize = 169;

我感觉这两个东西应该是一回事吧，都是169

也就是说会有一个变量在存储调用次数这个值，然后每次调用的时候加1

可能是在match之前进行维护这个值
pub fn syscall(syscall_id: usize, args: [usize; 3]) -> isize {
    match syscall_id {
        SYSCALL_WRITE => sys_write(args[0], args[1] as *const u8, args[2]),
        SYSCALL_EXIT => sys_exit(args[0] as i32),
        SYSCALL_YIELD => sys_yield(),
        SYSCALL_GET_TIME => sys_get_time(args[0] as *mut TimeVal, args[1]),
        SYSCALL_TASK_INFO => sys_task_info(args[0] as *mut TaskInfo),
        _ => panic!("Unsupported syscall_id: {}", syscall_id),
    }
}

提示：系统调用次数可以考虑在进入内核态系统调用异常处理函数之后，进入具体系统调用函数之前维护。

看到这个提示，我内心是崩溃的，啥叫内核态系统？异常处理函数又在哪里

理解起来有点棘手，问一下gpt：

1. **内核态（Kernel Mode）和用户态（User Mode）**：操作系统运行在两种不同的权限级别下：内核态和用户态。在内核态，操作系统具有对系统硬件和资源的完全访问权限；而在用户态，应用程序只能访问受限资源。当应用程序需要执行特权操作（如访问硬件或执行系统调用）时，它会请求操作系统切换到内核态执行。

2. **异常处理函数**：异常是在程序执行过程中出现的特殊情况，如访问非法内存地址或除以零。当异常发生时，操作系统会中断当前程序的执行并转到异常处理函数。这个函数负责处理异常情况，可能是修复问题或终止程序的执行。

按照我的理解就是在match之前，增加一个对syscall_id进行记录的过程。
如果在函数每次调用的时候，能够记录存储之前的syscall_id的值，并修改，那么这里应该有一个函数外部的变量。

没有好思路，我决定从头阅读代码，看了一下文件头部有一堆英文注释。翻译一下
所有系统调用的单一入口点是 [`syscall()`]，当用户空间希望使用 `ecall` 指令执行系统调用时会调用它。在这种情况下，处理器会引发一个“U模式环境调用”异常，该异常在 [`crate::trap::trap_handler`] 中作为其中一种情况处理。

为了清晰起见，每个单一的系统调用都被实现为它自己的函数，命名为 `sys_` 然后是系统调用的名称。你可以在子模块中找到这样的函数，并且你也应该以这种方式实现系统调用。

继续阅读trap部分的注释
陷阱处理功能

对于 rCore，我们有一个单一的trap入口点，即 __alltraps。在 [init()] 中的初始化过程中，我们将 stvec CSR 设置为指向它。

所有trap都经过 __alltraps，它在 trap.S 中定义。汇编语言代码仅执行足够的工作来恢复内核空间上下文，确保 Rust 代码安全运行，并将控制传递给 [trap_handler()]。

然后，它根据异常的确切情况调用不同的功能。例如，定时器中断会触发任务抢占，而系统调用会转到 [syscall()]。


阅读 TaskManager 的实现，思考如何维护内核控制块信息（可以在控制块可变部分加入需要的信息）。

继续阅读代码

任务管理实现

所有关于任务管理的内容，如启动和切换任务，都在这里实现。

一个名为 `TASK_MANAGER` 的 [`TaskManager`] 的单一全局实例控制着操作系统中的所有任务。

当你在 `switch.S` 中看到 `__switch` 汇编函数时要小心。该函数周围的控制流可能不是你期望的。

看到这里就有思路了，一个全局对象来了。结合提示，应该需要维护内核控制块信息。
在控制块可变部分加入需要的信息
pub struct TaskControlBlock {
    /// The task status in it's lifecycle
    pub task_status: TaskStatus,
    /// The task context
    pub task_cx: TaskContext,
}

/// The task manager, where all the tasks are managed.
///
/// Functions implemented on `TaskManager` deals with all task state transitions
/// and task context switching. For convenience, you can find wrappers around it
/// in the module level.
///
/// Most of `TaskManager` are hidden behind the field `inner`, to defer
/// borrowing checks to runtime. You can see examples on how to use `inner` in
/// existing functions on `TaskManager`.

这里需要研究一下inner,根据这个例子，再实现几个fn?
let mut inner = self.inner.exclusive_access();
let current = inner.current_task;
这些是共同点，inner.tasks[current]能够获取当前task
在TaskControlBlock增加  syscall_times: [u32; MAX_SYSCALL_NUM]，用于记录系统调用次数

修改一下初始方法，可以参数user里的代码，
pub struct TaskInfo {
    pub status: TaskStatus,
    pub syscall_times: [u32; MAX_SYSCALL_NUM],
    pub time: usize,
}

impl TaskInfo {
    pub fn new() -> Self {
        TaskInfo {
            status: TaskStatus::UnInit,
            syscall_times: [0; MAX_SYSCALL_NUM],
            time: 0,
        }
    }
}

看到此处，我发现这个结构体跟 TaskControlBlock 有点像啊，那么我再补上一个time

pub fn record_syscall_times(syscall_id: usize) {
    TASK_MANAGER.record_syscall_times(syscall_id);
}

突然发现这里有个红线
missing documentation for a function

补上注释，文档注释要用三个/,重要的事情说三遍？ task.rs中新增的两个属性，也要把注释写上

完成sys_task_info部分的代码，把系统调用次数取出来
task/mod.rs
``` rust
/// 获取当前调用次数
pub fn cur_syscall_times() ->[u32; MAX_SYSCALL_NUM] {
    TASK_MANAGER.inner.exclusive_access().tasks[TASK_MANAGER.inner.exclusive_access().current_task].syscall_times
}

/// 获取当前状态
pub fn cur_task_status() ->TaskStatus {
    TASK_MANAGER.inner.exclusive_access().tasks[TASK_MANAGER.inner.exclusive_access().current_task].task_status
}

/// 获取当前时间
pub fn cur_time() ->usize {
    TASK_MANAGER.inner.exclusive_access().tasks[TASK_MANAGER.inner.exclusive_access().current_task].time
}
```
already borrowed: BorrowMutError,这里有个错误，需要修改一下

syscall/process.rs
sys_task_info 函数
``` rust
if let Some(ti) = unsafe { ti.as_mut() } {
    ti.status = cur_task_status();
    ti.time = cur_time();
    ti.syscall_times = cur_syscall_times();
    0
} else {
    -1
}
```

[PASS] found <get_time OK1300019028970125670! (\d+)>
[FAIL] not found <Test sleep OK1300019028970125670!>
[PASS] found <current time_msec = (\d+)>
[PASS] found <time_msec = (\d+) after sleeping (\d+) ticks, delta = (\d+)ms!>
[PASS] found <Test sleep1 passed1300019028970125670!>
[FAIL] not found <string from task info test>
[FAIL] not found <Test task info OK1300019028970125670!>

错误又多了一个 T T
重新修改一下上面3个fn

/// 获取当前调用次数
pub fn cur_syscall_times() ->[u32; MAX_SYSCALL_NUM] {
    let inner = TASK_MANAGER.inner.exclusive_access();
    let current = inner.current_task;
    inner.tasks[current].syscall_times
}

/// 获取当前状态
pub fn cur_task_status() ->TaskStatus {
    let inner = TASK_MANAGER.inner.exclusive_access();
    let current = inner.current_task;
    inner.tasks[current].task_status
}

/// 获取当前时间
pub fn cur_time() ->usize {
    let inner = TASK_MANAGER.inner.exclusive_access();
    let current = inner.current_task;
    inner.tasks[current].time
}

再次运行
[PASS] found <get_time OK527815817627370! (\d+)>
[PASS] found <Test sleep OK527815817627370!>
[PASS] found <current time_msec = (\d+)>
[PASS] found <time_msec = (\d+) after sleeping (\d+) ticks, delta = (\d+)ms!>
[PASS] found <Test sleep1 passed527815817627370!>
[FAIL] not found <string from task info test>
[FAIL] not found <Test task info OK527815817627370!>

写了这么多没解决问题么，看看报错信息
Panicked at src/bin/ch3_taskinfo.rs:26, assertion failed: t2 - t1 <= info.time + 1
看看之前的报错信息
Panicked at src/bin/ch3_taskinfo.rs:21, assertion failed: 3 <= info.syscall_times[SYSCALL_GETTIMEOFDAY]

原来是在21行就panic了，现在走到26行了，time这个地方，刚刚仅仅是增加了time这个参数，但没有做任何处理
也就是说要至少要给time一个初始值，现在是0，肯定不对

运行时间 time 返回系统调用时刻距离任务第一次被调度时刻的时长，
也就是说这个时长可能包含该任务被其他任务抢占后的等待重新调度的时间。
程序运行时间可以通过调用 get_time() 获取，注意任务运行总时长的单位是 ms。

搜索了一下get_time,有个疑问，为什么不直接调用get_time_ms呢？

现在需要给time一个初始值，这个初始值应该是系统调用时刻距离任务第一次被调度时刻的时长

修改一下代码

fn record_syscall_times(syscall_id: usize) {
    let mut inner = TASK_MANAGER.inner.exclusive_access();
    let current = inner.current_task;
    inner.tasks[current].syscall_times[syscall_id] += 1;
    inner.tasks[current].time = get_time_ms();
}

重新运行了检查一下，居然全部通过了？

运行时间 time 返回系统调用时刻距离任务第一次被调度时刻的时长，
也就是说这个时长可能包含该任务被其他任务抢占后的等待重新调度的时间。

感觉这里不应该通过啊

不明白上面的写法为什么能通过，现在改了几次代码也通过测试了，如果能调试系统就好了