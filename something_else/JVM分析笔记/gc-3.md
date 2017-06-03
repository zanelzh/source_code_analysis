# 垃圾收集器

首先看一下各个收集器的位置以及相互的使用组合。
![](pic/gc.png)  

- Serial收集器
serial收集器是一个单线程的收集器，进行垃圾收集时，必须暂停其他工作线程。  
优点：简单高效。对于运行在Client模式下的虚拟机来说是一个很好的选择。
![](pic/serial.png)

- ParNew收集器
ParNew收集器其实就是Serial收集器的多线程版本，它是许多运行在Server模式下的虚拟机中首选的新生代收集器，其中有一个与性能无关但很重要的原因是，除了Serial收集器以外，目前只有它能与CMS收集器配合工作。
![](pic/parnew.png)
- Parallel Scavenge收集器


- Serial Old收集器


- Parallel Old收集器


- CMS收集器


- G1收集器

