# 9.1 设备中断处理：

Many device drivers execute code in two contexts: a ***top half*** that runs in a process’s kernel thread, and a ***bottom half*** that executes at interrupt time. The top half is called via system calls such as read and write that want the device to perform I/O. This code may ask the hardware to start an operation (e.g., ask the disk to read a block); then the code waits for the operation to complete. Eventually the device completes the operation and raises an interrupt. The driver’s interrupt handler, acting as the bottom half, figures out what operation has completed, wakes up a waiting process if appropriate, and tells the hardware to start work on any waiting next operation.

分两部分：

## Top half

运行在 process  的内核中，例如执行write read 等操作，然后向设备发一个操作指令，等待操作完成

## bottom half

运行在中断时，当设备完成kernel的操作后，raise a interrupt, 通知操作系统操作已经完成，唤醒等待的进程，来继续等待着的未完成的操作

