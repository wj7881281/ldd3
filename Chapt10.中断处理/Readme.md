尽管有些设备只使用其 I/O 区域即可进行控制，但大多数实际设备都比这要复杂一些。设备必须与外部世界打交道，其中通常包括旋转磁盘、移动磁带、连接远方的电线等。许多工作必须在与处理器不同且慢得多的时间范围内完成。由于几乎总是不希望让处理器等待外部事件，因此设备必须有一种方法让处理器知道发生了什么事。

当然，这种方式就是中断。中断只是硬件在需要处理器注意时可以发送的信号。Linux 处理中断的方式与处理用户空间信号的方式大致相同。大多数情况下，驱动程序只需为其设备的中断注册一个处理程序，并在中断到达时正确处理它们。当然，在这幅简单的图景之下，隐藏着一些复杂性。特别是，由于中断处理程序的运行方式，它们可以执行的操作在某种程度上受到限制。

如果没有真正的硬件设备来生成中断，则很难演示中断的使用。因此，本章中使用的示例代码适用于并行端口。此类端口在现代硬件上开始变得稀缺，但幸运的是，大多数人仍然能够使用具有可用端口的系统。我们将使用上一章中的简短模块；通过一些小的添加，它可以生成并处理来自并行端口的中断。该模块的名称short实际上意味着short int（它是C语言，不是吗？），以提醒我们它处理中断。

然而，在我们进入主题之前，是时候提出一个警告了。中断处理程序本质上是与其他代码同时运行的。因此，它们不可避免地会引发数据结构和硬件的并发和争用问题。如果您经不住诱惑而跳过第五章中的讨论，我们理解。但我们也建议您现在回头再看一下。在处理中断时，对并发控制技术的深入理解至关重要。

