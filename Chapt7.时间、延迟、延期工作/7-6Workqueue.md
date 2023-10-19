从表面上看，工作队列类似于微线程；它们允许内核代码请求在将来的某个时间调用某个函数。然而，两者之间存在一些显着差异，包括：
- Tasklet 在软件中断上下文中运行，因此所有 Tasklet 代码都必须是原子的。相反，工作队列函数在特殊的内核进程的上下文中运行；因此，他们拥有更大的灵活性。特别是，工作队列函数可以休眠。
- Tasklet 始终在最初提交它们的处理器上运行。默认情况下，工作队列以相同的方式工作。
- 内核代码可以请求将工作队列函数的执行延迟明确的时间间隔。

两者之间的主要区别在于，tasklet 在短时间内以原子模式快速执行，而工作队列函数可能具有较高的延迟，但不一定是原子的。每种机制都有适合的情况。

工作队列有一个 struct workqueue_struct 类型，它在 linux/workqueue.h 中定义。工作队列必须在使用前显式创建，使用以下两个函数之一：

```c
struct workqueue_struct *create_workqueue(const char *name);
struct workqueue_struct *create_singlethread_workqueue(const char *name);
```
每个工作队列都有一个或多个专用进程（“内核线程”），它们运行提交到队列的函数。如果使用 create_workqueue，您将获得一个工作队列，该工作队列为系统上的每个处理器都有一个专用线程。

要将任务提交到工作队列，您需要填写一个work_struct结构。这可以在编译时完成，如下所示：
```c
DECLARE_WORK(name, void (*function)(void *), void *data);
```

其中 name 是要声明的结构的名称，function 是要从工作队列调用的函数，data 是要传递给该函数的值。如果需要在运行时设置work_struct结构，请使用以下两个宏：
```c
INIT_WORK(struct work_struct *work, void (*function)(void *), void *data);
PREPARE_WORK(struct work_struct *work, void (*function)(void *), void *data);
```
INIT_WORK 在初始化结构方面做了更彻底的工作；您应该在第一次设置该结构时使用它。PREPARE_WORK 执行几乎相同的工作，但它不会初始化用于将 work_struct 结构链接到工作队列的指针。如果该结构当前可能已提交到工作队列，并且您需要更改该结构，请使用 PREPARE_WORK 而不是 INIT_WORK。

有两个函数可以将工作提交到工作队列：
```c
int queue_work(struct workqueue_struct *queue, struct work_struct *work);
int queue_delayed_work(struct workqueue_struct *queue, 
                       struct work_struct *work, unsigned long delay);
```
任一者都会将工作添加到给定队列中。然而，如果使用queue_delayed_work，则至少在延迟jiffies过去之后才执行实际工作。如果工作已成功添加到队列，则这些函数的返回值为 0；非零结果意味着该 work_struct 结构已经在队列中等待，并不需要第二次添加。

在未来的某个时间，将使用给定的数据值调用工作函数。该函数将在工作线程的上下文中运行，因此它可以在需要时休眠 - 尽管您应该知道该休眠可能会如何影响提交到同一工作队列的任何其他任务。然而，该函数不能做的是访问用户空间。由于它在内核线程内运行，因此根本没有可访问的用户空间。

如果您需要取消待处理的工作队列条目，您可以调用：
```c
int cancel_delayed_work(struct work_struct *work);
```
如果条目在开始执行之前被取消，则返回值非零。内核保证在调用cancel_delayed_work之后不会启动给定条目的执行。然而，如果cancel_delayed_work 返回0，则该条目可能已经在不同的处理器上运行，并且在调用cancel_delayed_work 后可能仍在运行。要绝对确保在 cancel_delayed_work 返回 0 后工作函数不会在系统中的任何位置运行，您必须在该调用之后调用：
```c
void flush_workqueue(struct workqueue_struct *queue);
```
在flush_workqueue返回后，调用之前提交的工作函数不会在系统中的任何地方运行。

当你完成工作队列后，你可以通过以下方式删除它：
```c
void destroy_workqueue(struct workqueue_struct *queue);
```

### 7.6.1共享队列
在许多情况下，设备驱动程序不需要自己的工作队列。如果您只是偶尔向队列提交任务，那么简单地使用内核提供的共享默认工作队列可能会更有效。但是，如果您使用此队列，您必须意识到您将与其他人共享它。除此之外，这意味着您不应该长时间独占队列（不长时间休眠），并且您的任务可能需要更长的时间才能轮到处理器。
jiq（“刚刚在队列中”）模块导出两个文件，演示共享工作队列的使用。它们使用单​​个 work_struct 结构，其设置方式如下：
```c
static struct work_struct jiq_work;

    /* this line is in jiq_init(  ) */
    INIT_WORK(&jiq_work, jiq_print_wq, &jiq_data);
```
当进程读取 /proc/jiqwq 时，模块会立即通过共享工作队列启动一系列行程。它使用的函数是：
```c
int schedule_work(struct work_struct *work);
```
请注意，在使用共享队列时使用不同的函数；它只需要 work_struct 结构作为参数。 jiq 中的实际代码如下所示：
```c
prepare_to_wait(&jiq_wait, &wait, TASK_INTERRUPTIBLE);
schedule_work(&jiq_work);
schedule(  );
finish_wait(&jiq_wait, &wait);
```
实际的工作函数就像 jit 模块一样打印出一行，然后，如果需要，将 work_struct 结构重新提交到工作队列中。这是 jiq_print_wq 的完整内容：
```c
static void jiq_print_wq(void *ptr)
{
    struct clientdata *data = (struct clientdata *) ptr;
    
    if (! jiq_print (ptr))
        return;
    
    if (data->delay)
        schedule_delayed_work(&jiq_work, data->delay);
    else
        schedule_work(&jiq_work);
}
```
如果用户正在读取延迟设备（/proc/jiqwqdelay），工作函数会使用schedule_delayed_work以延迟模式重新提交自身：
```c
int schedule_delayed_work(struct work_struct *work, unsigned long delay);
```
如果您查看这两个设备的输出，它看起来像：
```
% cat /proc/jiqwq
    time  delta preempt   pid cpu command
  1113043     0       0     7   1 events/1
  1113043     0       0     7   1 events/1
  1113043     0       0     7   1 events/1
  1113043     0       0     7   1 events/1
  1113043     0       0     7   1 events/1
% cat /proc/jiqwqdelay
    time  delta preempt   pid cpu command
  1122066     1       0     6   0 events/0
  1122067     1       0     6   0 events/0
  1122068     1       0     6   0 events/0
  1122069     1       0     6   0 events/0
  1122070     1       0     6   0 events/0
```
读取/proc/jiqwq时，每行打印之间没有明显的延迟。相反，当读取 /proc/jiqwqdelay 时，每行之间存在正好一 jiffy 的延迟。无论哪种情况，我们都会看到打印相同的进程名称；它是实现共享工作队列的内核线程的名称。CPU编号打印在斜杠之后；我们永远不知道读取 /proc 文件时哪个 CPU 将运行，但此后工作函数将始终在同一处理器上运行。

如果需要取消提交到共享队列的工作条目，可以使用cancel_delayed_work，如上所述。然而，刷新共享工作队列需要一个单独的函数：
```c
void flush_scheduled_work(void);
```
由于您不知道还有谁可能正在使用此队列，因此您永远不知道需要多长时间才能返回flush_scheduled_work。



