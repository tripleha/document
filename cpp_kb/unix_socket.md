## unix socket 系统调用学习(2.6.32)

    socket 是 系统提供的一层统一接口，用于不同主机间进程间通信的手段，也可用于本地进程通信，
        其本质实际上是一层特殊的文件系统，连接用户进程和内核协议栈。
        (注册文件系统#)

    sys/socket.h 头文件
    netinet/in.h 头文件 定义了和地址相关的结构体

    int socket(int __domain, int __type, int __protocol) 调用
        参数解析
            int __domain 指定 协议簇，使用socket.h中的宏定义 Protocol families PF_ 开头
                PF_INET ip协议
                PF_INET6 ip协议 ipv6版
            int __type 指定 协议簇下的协议类型
                SOCK_STREAM 数据流式 TCP
                SOCK_DGRAM 数据包式 UDP
            int __protocol 指定 具体协议类型
                IPPROTO_IP 0
                IPPROTO_TCP 6
                IPPROTO_UDP 17
        返回结果
            创建socket所绑定的文件描述符
            -1 如果创建失败则返回该值

    TCP socket建立的过程
        sys_sock  系统调用入口
            sock_create  创建一个 struct socket 结构
                sock_alloc  分配 struct socket 空间
                *inet_create  创建 struct sock 结构并初始化
                    sk_alloc  分配 struct sock 空间 实际分配的是 tcp_sock 的空间
                    sock_init_data  初始化 struct sock 相关字段
                    *tcp_v4_init_sock  初始化 inetconnection_sock 和 tcp_sock 相关字段
            sock_map_fd  为 struct socket 结构绑定文件
                sock_alloc_fd  获取新的文件描述符(fd) 和 struct file 结构
                sock_attach_fd  让 struct socket 和 前面分配的 struct file 结构关联起来
                fd_install  让新分配 fd 和 struct file 关联起来
    
    三个 struct socket sock file
        socket 提供给外部(内核的其他模块)操作的，结构简单
        sock 与具体协议栈的实现有关，不同的不一样，是内核socket的具体实现
        file 对于用户而言的结构，用户操作socket就是在操作文件，所以file是socket的面向文件系统的结构
    这三个结构都是一一对应的
    当然最终用户获得的是文件描述符，每个进程的进程描述符中都可以找到文件描述符 fd 和 struct file 之间的映射表

    sock inet_sock inet_connection_sock tcp_sock 之间的关系:
        每个结构体的第一个成员变量必是上一级结构体，如此便可向上级转换，是一种变相的继承方式

    int setsockopt(int __fd, int __level, int __optname, const void *__optval, socklen_t __optlen) 调用

        几个常用 __optname:
        level = SOL_SOCKET:
            SO_REUSEADDR 复用地址 快速重启服务时，由于 TIME_WAIT 状态的存在，
                原来的socket还存在，会出现地址被占用的情况，使bind操作失败，所以使用这个选项来避免
            SO_REUSEPORT 复用端口 可以多个socket同时监听一个端口
        level = IPPROTO_TCP:
            TCP_NODELAY 用于禁止TCP协议的Nagle算法，使小数据包（小于最大数据段大小）立即发送，
                会降低TCP协议的效率，但在一些场合适用

    int bind(int __fd, const sockaddr *__addr, socklen_t __len) 调用
        将socket文件描述符和具体的协议地址绑定，
        地址的信息储存在 __addr 指向结构体中，根据不同协议会有不同形式，所以定义一个sockaddr用于传参
        返回结果
            0 成功
            -1 失败
        这一步其实可以不用去做，只不过如果我们希望指定的时候我们才会调用bind去指定，
        当我们没有调用bind的时候就调用connect或者listen时，系统会随机为我们分配一个端口和地址
        当地址为0，或者port为0时，也是随机分配的。

    bind函数内核动作:
        sys_bind
            sockfd_lookup_light  由fd找到对应socket结构体
                fget_light  通过fd找到file
                sock_from_file  通过file找到socket
            move_addr_to_kernel  将用户提供的地址信息(struct sockaddr)复制到内核空间
            *inet_bind  进行实际的地址绑定
                inet_addr_type  检查地址类型(然后进行类型转换，不同协议处理方式应该不同)
                地址赋值(未进行函数调用，而是直接操作)
                *inet_csk_get_port  设置端口

    inet_hashinfo 三个hash表的集合
        ehash -> 用于索引 TCP_ESTABLISHED <= sk->sk_state < TCP_CLOSE 的 sock
        bhash -> 用于索引绑定本地地址的 sock
        listening_hash 用于索引 sk->sk_state 为 TCP_LISTEN 状态的 sock

        ehash 和 bhash 是动态分配 listening_hash 是静态分配的数组
        它们创建和初始化过程 调用队列 fs_initcall(inet_init) => inet_init() => tcp_init()

        bhash 结构相关变量
            struct inet_bind_hashbucket *bhash;  // hash表节点数组
            unsigned int bhash_size;  // 记录hash表的长度，因为是动态分配的
            struct kmem_cache *bind_bucket_cachep;  // SLAB 高速缓存

            struct inet_bind_hashbucket {
                spinlock_t lock;
                struct hlist_head chain;
            };
            
            // chain 用于链接具有相同hash值的hash元素 inet_bind_bucket，他们内部通过 node 字段链接
            // owners 表示和 port 端口绑定的所有 sock

            struct inet_bind_bucket {
            #ifdef CONFIG_NET_NS
                struct net  *ib_net;
            #endif
                unsigned short  port;
                signed short  fastreuse;
                int  num_owners;
                struct hlist_node  node;
                struct hlist_head  owners;
            };
        
        随机端口选择过程(若传入port为0，则由系统随机分配一个可用的port)
            1.inet_get_local_port_range 取得可用端口的区间，然后随机选择其中一个
            2.找到端口对应的inet_bind_hashbucket，遍历其chain，若无元素或者有元素可以重用，则把这个端口作为候选端口，进入4，否则进入3
            3.递增随机端口，若超过上界则设为下界，递减可用端口数量，若可用端口数量为0则无可用端口失败返回，否则进入2
            4.若chain无元素则调用inet_bind_bucket_creat新建一个，使用SLAB高速缓存
            5.调用inet_bind_hash绑定端口，即设置inet_bind_bucket的port字段，还有inet_sk(sk)的num字段，然后将sk加入owners队列中
            注:若指定了port则直接从4开始，不过需要先检查复用问题

    端口复用条件(按顺序检查)
        1.绑定到不同接口的socket可以共用本地端口
        2.如果所有的socket都设置了sk->sk_reuse，并且全部的state都不是TCP_LISTEN，则可共用该端口
            (检查时为了避免反复遍历owners，使用fastreuse来快速确定，该字段只在添加时设置，空时为真值)
        3.如果所有socket都绑定了各自不同的inet_sk(sk)->rcv_saddr的本地地址，则该端口可共用

    在经过端口绑定等操作后，tcp_socket的inet_sock设置了下列字段:
        inet->rcv_saddr = inet->saddr = addr->sin_addr.s_addr (传递进来的 IP)
        inet->num = snum (传递进来的端口)
        inet->sport = htons(inet->num) (转换了字节序的端口)
        inet->daddr = 0;
        inet->dport = 0;

    int getsockname(int __fd, struct sockaddr *__restrict__ __addr, socklen_t *__restrict__ __len) 调用
        通过socket文件描述符获取其地址信息，信息放入__addr指向空间
        返回结果
            0 成功
            -1 失败

    int listen(int __fd, int __n) 调用
        监听socket，__n为最大半连接队列长度
        返回结果
            0 成功
            -1 失败

    listen函数内核动作:
        sys_listen
            sockfd_lookup_light  通过fd找到socket
            *inet_listen  处理socket监听
                inet_csk_listen_start
                    reqsk_queue_alloc  分配icsk的连接请求队列
                    *inet_csk_get_port  设置端口为不可重用
                    *inet_hash  把sock加入到监听队列中

    int accept(int __fd, sockaddr *__restrict__ __addr, socklen_t *__restrict__ __addr_len) 调用
        创建新的socket等待客户端连接到来，并在连接到来后完成连接，返回新socket的文件描述符
        当接受一个客户端连接时，__addr会被设置成该客户端的地址
        返回结果
            socket的fd
            -1 失败

    其实 accept 的作用就是有新的客户端连接到来的时候创建一个新的 socket 而已。
    sys_accept4 首先创建好 socket 结构，file 结构，并分配好文件描述符。
    之后获取 socket 结构对应的 sock 结构。
    形成一个完整的 socket 结构之后把 socket 文件描述符返回用户空间，
    这样服务器就可以通过这个新的 socket 和客户端进行通信了。

    int connect(int __fd, const struct sockaddr *__addr, socklen_t __len) 调用
        客户端socket通过该函数建立到服务端socket的连接
        返回结果
            0 成功
            -1 失败

    TCP 带外数据，所谓带外数据，一般是为了发送一些紧急信息。然而在TCP中，带外数据并非真的带外发送，是和普通数据一样遵循顺序的发送，只是会在TCP头设置进入紧急状态，使得接收方进行特殊处理，尽快接收数据。
