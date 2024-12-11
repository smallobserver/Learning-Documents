# Android内存泄露

触发OOM常见场景

- 内存抖动
- 内存泄露
- 文件数上限
- 线程数上限



## KOOM

周期轮询检测

- 内存占用率检测
- 线程数检测
- 文件描述符检测
- 内存增长检测

使用fork来在子进程中避免主进程卡死

fork在主进程下的某个线程调用，复制出来的子进程中，会有其他线程，但是仅仅是一个对象和状态没有start是个虚假的。如果这时候调用dumpHprofData(执行此方法会先suspendAll)就会死锁，无法得到返回值。避免这种情况，在调用fork前，主进程先suspendAll。

反射调用so中的方法

libart.so

   art::Dbg::SupsendVM

   art::Dbg::ResumeVM



hook  io操作，读取到写入hprof文件后，