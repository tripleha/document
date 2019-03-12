# reactor模式IO实现学习

    libev源码还是很厉害的，就是宏定义太多，我这种半吊子还是看看别人的解析好了

## libev

    watcher
        事件的监视器，一组事件监视器是以单向链表形式来组织的
        区分优先级，区分类别，记录对应回调函数和用户附加相关数据
        pending
            其用于索引监视器对应 anpending 在 anpending 数组中的位置

    anfd
        文件描述符的扩展结构，用于记录文件描述符绑定的事件和相关监视器
        主要数据: event 和 watcher链表头节点

    anfds
        采用连续内存记录anfd，实际上是一个anfd数组(逐渐扩展的)，以文件描述符作为下标
        其组织形式并未不能快速按照优先级排列
            所以增加了pending相关结构用于按优先级执行就绪的监视器的回调

    ev_loop
        该结构主要是用于不同线程保存自己的事件循环变量的，
        按优先级执行相关数据结构
        pendings[]
            一个定长数组，记录的是anpending数组的头指针
            从后往前遍历 pendings 即可按优先级遍历就绪监视器
        pendingcnt[]
            每个等级中当前有效的监视器个数
        pendingmax[]
            每个等级中已经记录的监视器个数

    anpending
        该结构包含 watcher指针 和 事件
        其数组空间的大小也是动态分配的，其是不缩小的，
            当一个 anpending 的对应回调执行后，其会减去 pendingcnt，如此从后往前执行。

    ev_run
