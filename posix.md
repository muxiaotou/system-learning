    https://blog.csdn.net/wang_hufeng/article/details/52317787  
    https://mp.weixin.qq.com/s?__biz=Mzg3NTU3OTgxOA==&mid=2247489085&idx=1&sn=1eb51ea0a00a00a7a62f232221bafaf8&chksm=cf3e06f8f8498fee063670f21bd0a756ef821e359709ed62720b48927ad18eeb0757399036a8&cur_album_id=1778430570125967361&scene=190#rd

    应用程序通过应用编程接口(API)而不是直接通过系统调用来编程。一个API定义了一组应用程序使用的编程接口，即可以实现一个系统调用，
    也可以通过调用多个系统调用来实现。

    posix就是一种操作系统的接口标准，linux、unix、windows都遵循这个规范。
    posix仅仅定义了一个接口，而非一种实现，所以并不区分系统调用和库函数。

    系统调用时为了方便使用操作系统的接口，而库函数时为了人们编程的方便，各种语言的库函数都是基于系统调用之上的。
    比如c语言的libc库，go语言的os、syscall库都是库函数，都是对系统调用的封装，屏蔽系统细节，为跨平台提供基础。

    简单总结：
    完成同一功能，不同内核提供的系统调用（也就是一个函数）是不同的，例如创建进程，linux下是fork函数，windows下是creatprocess函数。好，我现在在
    linux下写一个程序，用到fork函数，那么这个程序该怎么往windows上移植？我需要把源代码里的fork通通改成creatprocess，然后重新编译！
    
    posix标准的出现就是为了解决这个问题。linux和windows都要实现基本的posix标准，linux把fork函数封装成posix_fork（随便说的），windows把
    creatprocess函数也封装成posix_fork，都声明在unistd.h里。
    这样，程序员编写普通应用时候，只用包含unistd.h，调用posix_fork函数，程序就在源代码级别可移植了。

    系统调用是内核代码，内核代码能访问系统上的所有地址空间，而我们执行的代码是用户空间的代码，常用的系统调用：
    open close read write ioctl fork clone exit getpid 等，需要包含unistd.h 等头文件

    库函数时高层的，在系统调用上的一层包装，运行在用户态，为程序员提供真正的幕后完成实际事务系统调用的更为方便的接口。
    printf scanf fopen fclose 等，需要包含stdio.h stdlib.h等头文件

    库函数时语言本身的一部分，系统调用时内核提供给应用程序的接口，属于系统的一部分。

    库函数和系统调用区别：
    系统调用是各个系统实现的，因此各个系统的系统调用是不同的，而比如c语言的库函数在各个系统是相同的，库函数是夸平台的，库函数的存在也是为了免除使用
    应用程序直接使用系统调用带来的跨平台差异