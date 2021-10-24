# goroutine

Golang 在语言级别支持协程，称之为 goroutine。Golang 标准库提供的所有系统调用操作(包括所有的同步 IO 操作)，都会出让 CPU 给其他 goroutine。这让 goroutine 的切换管理不依赖于系统的线程和进程，也不依赖于 CPU 的核心数量，**而是交给 Golang 的运行时 runtime 统一调度。**