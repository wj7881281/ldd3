### 开头
内核编程带来了其独特的调试挑战。内核代码不能在调试器下轻松执行，也不能轻松跟踪，因为它是一组与特定进程无关的功能。内核代码错误也可能非常难以重现，并且可能导致整个系统瘫痪，从而破坏了许多可用于追踪它们的证据

本章介绍了在这种困难的情况下可以用来监视内核代码和跟踪错误的技术

### 4.1内核所支持的仿真技术
在第 2 章中，我们建议您构建并安装自己的内核，而不是运行发行版附带的库存内核。运行自己的内核的最重要原因之一是内核开发人员已在内核本身中内置了多个调试功能。这些功能会产生额外的输出并降低性能，因此它们往往不会在发行商的生产内核中启用。然而，作为内核开发人员，您有不同的优先级，并且会很乐意接受额外内核调试支持的（最小）开销

在这里，我们列出了应为用于开发的内核启用的配置选项。除非另有说明，所有这些选项都可以在您喜欢的任何内核配置工具的“make menuconfig”菜单下找到。请注意，并非所有体系结构都支持其中一些选项。

CONFIG_DEBUG_KERNEL:
该选项只是使其他调试选项可用；它应该被打开，但本身并不启用任何功能,此config是仿真的基础开关，所有仿真功能必须打开这个开关

CONFIG_DEBUG_SLAB：
