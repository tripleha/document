# linux IO复用模型

    select, poll, epoll
    windows 下还有 iocp

## select使用

    <sys/select.h> 头文件

    void FD_SET(int fd, fd_set *set)
        在fd_set中设置要监听的文件描述符所对应位为1
    void FD_ZERO(fd_set *set)
        清零 fd_set 的所有位
    void FD_CLR(int fd, fd_set *set)
        将文件描述符在fd_set中对应位 置零
    int  FD_ISSET(int fd, fd_set *set)
        查询fd_set中文件描述符对应位是否被设置为1

    int select(int nfds, fd_set *readfds, fd_set *writefds,
        fd_set *exceptfds, struct timeval *timeout)
        超时精度是微秒级的，select可能更新timeout参数来表示剩余时间
        当fd_set中有文件描述符发生对应事件时(有数据读或者有空间写)返回
            将对应fd_set中发生事件的位 置1，之后只需遍历相关的fd_set即可知道

    int pselect(int nfds, fd_set *readfds, fd_set *writefds,
        fd_set *exceptfds, const struct timespec *timeout,
        const sigset_t *sigmask)
        与 select() 相比，其所使用的 timeout 更加精确(纳秒级)，且不会改变timeout

    FD_SETSIZE
        select 只能处理小于 FD_SETSIZE 大小的数量的 文件描述符
        且只能处理 0~FD_SETSIZE 之间的文件描述符
    fd_set 结构体
        用于储存文件描述符状态是否有相应事件发生的结构体
        文件描述符在其中的组织方式是以数组形式组织的
            为了节省空间，采用 long 的每一位来表示每个文件描绘符

## poll使用


## select和poll的内核实现细节

    struct file_operations 结构，定义了各种用于操作设备的函数指针。(用 f_op 表示)
        指向操作每个文件设备的驱动程序实现的具体操作函数，即设备驱动的回调函数。
        通过这个结构体，linux内核就不需要知道每个具体的文件设备是如何操作的。
        这个结构体是与file结构绑定的。
        定义了 poll 方法

    wait_queue 结构

    select和poll的实现基本是一样的，只是传递参数有所不同。

    基本步骤都是遍历

## epoll
