另一个与计时问题相关的内核设施是tasklet 机制。它主要用于中断管理（我们将在第 10 章再次看到它。）
Tasklet 在某些方面类似于内核定时器。它们总是在中断时运行，它们总是在调度它们的同一个 CPU 上运行，并且它们接收一个 unsigned long 参数。然而，与内核计时器不同的是，您不能要求在特定时间执行该函数。通过调度一个tasklet，您只需要求它在内核选择的稍后时间执行即可。此行为对于中断处理程序特别有用，其中必须尽快管理硬件中断，但大多数数据管理可以安全地延迟到稍后的时间。实际上，tasklet 就像内核计时器一样，在“软中断”上下文中执行（以原子模式），“软中断”是一种在启用硬件中断的情况下执行异步任务的内核机制。
微线程作为一种数据结构存在，在使用之前必须对其进行初始化。可以通过调用特定函数或使用某些宏声明结构来执行初始化：
```c
#include <linux/interrupt.h>

struct tasklet_struct {
      /* ... */
      void (*func)(unsigned long);
      unsigned long data;
};

void tasklet_init(struct tasklet_struct *t,
      void (*func)(unsigned long), unsigned long data);
DECLARE_TASKLET(name, func, data);
DECLARE_TASKLET_DISABLED(name, func, data);
```
Tasklet 提供了许多有趣的功能：
- 可以禁用并稍后重新启用微线程；只有在启用和禁用次数相同的情况下才会执行它。

- 就像定时器一样，tasklet 可以重新注册自己。
- 可以将微线程安排为以正常优先级或高优先级执行。后一组始终首先执行。
- 一个tasklet 可以与其他tasklet 并发，但相对于其自身是顺序执行——同一个tasklet 绝不会同时在多个处理器上运行。此外，如前所述，tasklet 始终在调度它的同一个 CPU 上运行。

jit 模块包含两个文件，/proc/jitasklet 和 /proc/jitasklethi，它们返回与第 7.4 节中介绍的 /proc/jitimer 相同的数据。当您读取其中一个文件时，您会得到一个标头和六个数据行。第一个数据行描述调用进程的上下文，其他行描述微线程过程的连续运行的上下文。这是编译内核时运行的示例：
```
phon% cat /proc/jitasklet
   time   delta  inirq    pid   cpu command
  6076139    0     0      4370   0   cat
  6076140    1     1      4368   0   cc1
  6076141    1     1      4368   0   cc1
  6076141    0     1         2   0   ksoftirqd/0
  6076141    0     1         2   0   ksoftirqd/0
  6076141    0     1         2   0   ksoftirqd/0
```
从上面的数据可以看出，只要 CPU 正忙于运行进程，tasklet 就会在下一个计时器滴答处运行，但当 CPU 空闲时，它会立即运行。内核提供了一组 ksoftirqd 内核线程，每个 CPU 一个，只是为了运行“软中断”处理程序，例如 tasklet_action 函数。因此，tasklet 的最后三次运行发生在与 CPU 0 关联的 ksoftirqd 内核线程的上下文中。jitasklethi 实现使用高优先级的 tasklet，这将在即将发布的函数列表中进行解释。

jit 中实现 /proc/jitasklet 和 /proc/jitasklethi 的实际代码与实现 /proc/jitimer 的代码几乎相同，但它使用 tasklet 调用而不是计时器调用。下面的列表详细列出了微线程结构初始化后的内核接口：

_void tasklet_disable(struct tasklet_struct *t);_
该函数禁用给定的tasklet。该tasklet 仍可以使用tasklet_schedule 进行调度，但其执行会被推迟，直到再次启用该tasklet。如果tasklet当前正在运行，则该函数忙等待直到tasklet退出；因此，在调用tasklet_disable之后，您可以确定tasklet没有在系统中的任何地方运行。

_void tasklet_disable_nosync(struct tasklet_struct *t);_
禁用该tasklet，但不等待任何当前正在运行的函数退出。当它返回时，tasklet 被禁用，并且在重新启用之前不会被调度，但当函数返回时它可能仍在另一个 CPU 上运行。

_void tasklet_schedule(struct tasklet_struct *t);_
安排tasklet 的执行。如果一个tasklet在有机会运行之前被再次调度，它只运行一次。但是，如果在运行时安排它，则它会在完成后再次运行；这确保了在处理其他事件时发生的事件得到应有的关注。这确保了在处理其他事件时发生的事件得到应有的关注。

_void tasklet_hi_schedule(struct tasklet_struct *t);_
安排tasklet 以更高的优先级执行。当软中断处理程序运行时，它会先处理高优先级的微线程，然后再处理其他软中断任务，包括“普通”微线程。理想情况下，只有具有低延迟要求的任务（例如填充音频缓冲区）才应使用此函数，以避免其他软中断处理程序引入的额外延迟。实际上，/proc/jitasklethi 与 /proc/jitasklet 没有人类可见的差异。

_void tasklet_kill(struct tasklet_struct *t);_
该函数确保tasklet不会被调度再次运行；通常在关闭设备或删除模块时调用它。如果计划运行该微线程，则该函数将等待直至其执行。如果tasklet自行重新调度，则必须在调用tasklet_kill之前阻止它自行重新调度，就像del_timer_sync一样。

Tasklet 在 kernel/softirq.c 中实现。两个tasklet列表（普通和高优先级）被声明为每个CPU的数据结构，使用与内核定时器相同的CPU亲和性机制。小任务管理中使用的数据结构是一个简单的链表，因为小任务没有内核定时器的排序要求。