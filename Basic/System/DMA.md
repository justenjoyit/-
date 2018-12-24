# DMA

> https://baike.baidu.com/item/DMA/2385376?fr=aladdin

DMA(Direct Memory Access，直接内存访问)，允许不同速度的硬件装置来沟通，而不需要依赖于CPU的大量中断负载。否则，CPU需要从来源把每一片段的资料复制到暂存器，然后把它们再次写回到新的地方。在这个过程中，CPU对于其他的工作来说就无法使用。

在实现DMA传输时，是由DMA控制器直接掌管总线，因此，存在着一个总线控制权转移。即DMA传输前，CPU要把总线控制权交给DMA控制器，而在结束DMA传输后，DMA控制器应立即把总线控制权再交给CPU。

一个完整的DMA传输过程必须经过DMA请求、DMA响应、DMA传输、DMA结束4个步骤。