# 🚀 服务器并发模型与IO多路复用

> 📌 适用场景：嵌入式Linux高并发服务器、物联网网关、多设备管理、实时数据转发

[TOC]

---

## 一、服务器模型对比

### 1.1 单循环服务器（迭代服务器）

```
📦 核心特点：同一时刻只能服务一个客户端，处理完才能响应下一个

🔄 工作流程（以TCP为例）：
    Server
       │
       │  socket() → bind() → listen()
       │
       │  ◄── 客户端A连接 ──► accept() → conn_sock_A
       │                      │
       │                      ├─ recv() 接收数据
       │                      ├─ 业务处理（可能阻塞！）
       │                      ├─ send() 回复响应
       │                      │
       │                      close(conn_sock_A)
       │
       │  ◄── 客户端B连接 ──► accept() → conn_sock_B  ← B必须等A处理完！
       │                      │
       │                      ...（同上）

✅ 优点：
   - 代码简单，逻辑清晰，易调试
   - 无并发竞争，无需锁机制
   - 资源占用少（单进程/单线程）

❌ 缺点：
   - ❌ 客户端必须串行等待，实时性差
   - ❌ 业务处理阻塞时，其他客户端完全无法连接
   - ❌ 不适合多设备并发场景

🎯 嵌入式适用场景：
   - 调试阶段/原型验证
   - 设备只连接单一上位机（如配置工具）
   - 业务逻辑极简单（<10ms处理时间）
```

### 1.2 并发服务器模型

```
📦 核心目标：同一时刻响应多个客户端请求

┌─────────────────────────────────────────┐
│ 方案1：多进程（fork）                     │
├─────────────────────────────────────────┤
│ 🔹 流程：                                │
│   主进程accept() → fork()子进程 → 子进程处理 │
│ 🔹 优点：进程隔离，一个崩溃不影响其他        │
│ 🔹 缺点：进程创建开销大，IPC通信复杂         │
│ 🔹 嵌入式：资源消耗大，慎用（>10并发不推荐）  │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ 方案2：多线程（pthread）                   │
├─────────────────────────────────────────┤
│ 🔹 流程：                                │
│   主线程accept() → pthread_create() → 子线程处理 │
│ 🔹 优点：线程切换快，共享内存通信方便         │
│ 🔹 缺点：需加锁防竞争，一个线程崩溃可能影响整体 │
│ 🔹 嵌入式：中等资源设备可用（注意栈大小+互斥锁）│
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ 方案3：IO多路复用（select/poll/epoll）⭐│
├─────────────────────────────────────────┤
│ 🔹 流程：                                │
│   单进程单线程 + 事件循环 + 非阻塞IO        │
│   → 一个线程管理所有连接，谁就绪处理谁       │
│ 🔹 优点：                                │
│   - 系统开销最小，无进程/线程创建开销         │
│   - 适合高并发（1000+连接）                │
│   - 代码可控，无锁竞争问题              	   │
│ 🔹 缺点：                                │
│   - 编程模型复杂（状态机+回调）             │
│   - 业务逻辑不能阻塞（需异步化或线程池）  	 │
│ 🔹 嵌入式：⭐首选方案⭐ 资源友好+高并发    │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ 方案4：异步IO（AIO / io_uring）           │
├─────────────────────────────────────────┤
│ 🔹 流程：提交IO请求→内核完成→回调通知    │
│ 🔹 优点：真正的非阻塞，零拷贝潜力        │
│ 🔹 缺点：内核支持要求高，接口复杂        │
│ 🔹 嵌入式：现阶段不推荐（内核版本限制）   │
└─────────────────────────────────────────┘
```

### 1.3 模型选型决策树（嵌入式场景）

```
🎯 需求：设计嵌入式网络服务器
│
├─ 并发连接数预期？
│  ├─ ≤ 5个 → ✅ 单循环服务器（简单够用）
│  └─ > 5个 → 进入下一步
│
├─ 设备资源情况？
│  ├─ RAM < 64KB → ✅ IO多路复用 + 精简状态机
│  ├─ RAM 64KB~256KB → ✅ IO多路复用 + 固定连接池
│  └─ RAM > 256KB → 进入下一步
│
├─ 业务逻辑是否复杂/可能阻塞？
│  ├─ 是 → ✅ 多线程（每连接一线程）或 线程池+epoll
│  └─ 否 → ✅ epoll + 非阻塞IO（纯事件驱动）
│
├─ 是否需要进程级隔离（安全/稳定性）？
│  ├─ 是 → ⚠️ 多进程（资源允许时）
│  └─ 否 → ✅ 保持单进程方案
│
💡 终极建议：
   "嵌入式首选epoll+非阻塞" — 在资源允许时，优先用单进程+epoll实现高并发；
   业务复杂时，用"epoll主循环+工作线程池"混合架构。
```

---

## 二、Linux IO模型详解

### 2.1 五种IO模型对比

```
📦 核心概念：应用程序如何等待数据从内核缓冲区→用户缓冲区

┌─────────────────────────────────────────────────┐
│ 模型          │ 阻塞阶段      │ 拷贝阶段    │ 嵌入式适用性 │
├─────────────────────────────────────────────────┤
│ 阻塞IO        │ 等待数据+拷贝 │ 阻塞        │ ⭐ 简单场景   │
│ 非阻塞IO      │ 轮询等待      │ 阻塞        │ ⚠️ 配合select│
│ IO多路复用    │ select等阻塞  │ 阻塞        │ ⭐⭐⭐ 首选    │
│ 信号驱动IO    │ 信号通知      │ 阻塞        │ ❌ 复杂/少用  │
│ 异步IO(AIO)   │ 完全异步      │ 异步        │ ❌ 内核要求高 │
└─────────────────────────────────────────────────┘
```

#### 🔹 阻塞IO（默认模式）

```c
// ✅ 特点：调用后阻塞，直到数据就绪+拷贝完成
ssize_t n = recv(sock, buf, len, 0);  // 阻塞等待数据
scnaf/getchar/fgets/gets
		
read/recv/recvfrom

// ✅ 优点：
//    - 编程简单，逻辑线性
//    - 适合单连接或业务简单的场景

// ❌ 缺点：
//    - 一个连接阻塞，整个进程/线程卡住
//    - 无法同时监控多个IO事件
1.可以实现多任务同步（多个事件相互影响）
2.可以节省CPU资源开销,提高执行效率
```

#### 🔹 非阻塞IO（轮询模式）

```c
// ✅ 设置非阻塞
int flags = fcntl(sock, F_GETFL, 0);
fcntl(sock, F_SETFL, flags | O_NONBLOCK);

// ✅ 特点：调用立即返回，无数据时返回-1 + EAGAIN/EWOULDBLOCK
ssize_t n = recv(sock, buf, len, 0);
if (n < 0) {
    if (errno == EAGAIN || errno == EWOULDBLOCK) {
        // 暂无数据，稍后重试
    } else {
        // 真实错误
    }
}

// ✅ 优点：
//    - 不会阻塞，可轮询多个fd
//    - 实现简单（相比select/epoll）

// ❌ 缺点：
//    - 轮询浪费CPU（忙等待）
//    - 需手动管理重试逻辑

// 🎯 嵌入式适用：
//    - 配合定时器轮询（非高频）
//    - 作为select/epoll的底层基础（必须设为非阻塞）
1.可以访问多个IO事件
2.配合轮询操作，浪费CPU资源
```

#### 🔹 信号驱动IO（SIGIO）

```c
// ✅ 开启信号驱动
fcntl(sock, F_SETOWN, getpid());  // 设置接收信号的进程
int flags = fcntl(sock, F_GETFL, 0);
fcntl(sock, F_SETFL, flags | O_ASYNC);  // 开启异步通知

// 注册信号处理函数
void sigio_handler(int sig) {
    // 数据就绪，执行recv（此时不会阻塞）
    recv(sock, buf, len, 0);
}
signal(SIGIO, sigio_handler);

// ✅ 优点：
//    - 异步通知，无需轮询
//    - 理论上节省CPU

// ❌ 缺点：
//    - 信号处理函数限制多（只能调用异步信号安全函数）
//    - 信号可能丢失/合并，可靠性难保证
//    - 多fd管理复杂（信号不区分具体fd）

// 🎯 嵌入式适用：
//    - ❌ 不推荐！调试困难，生产环境慎用
1.实现异步IO操作，节省CPU开销
2.只能针对比较少的IO事件
```

#### 🔹 异步IO（AIO / io_uring）⭐未来方向

```
📦 真正异步：应用提交请求→内核完成所有操作→通知应用

✅ 优点：
   - 完全非阻塞，零等待
   - 潜在零拷贝优化（io_uring）

❌ 缺点：
   - Linux AIO接口不完善（主要支持磁盘IO）
   - io_uring需要内核5.1+，嵌入式内核往往较旧
   - 编程模型复杂，调试困难

🎯 嵌入式适用：
   - ❌ 现阶段不推荐（等待生态成熟）
   - ✅ 高端嵌入式平台（如Jetson + 新内核）可探索
```

---

## 三、IO多路复用三剑客：select / poll / epoll

![多路复用含义图解-Telephony_multiplexer_system](https://cdn.jsdelivr.net/gh/bingoteng/Photos/Typora/20260718_01-34_eb771c933aba04238980b504a53f59a2.gif)

### 3.1 核心思想对比

```
🎯 共同目标：用一个系统调用监控多个文件描述符，谁就绪就处理谁

┌─────────────────────────────────────────┐
│ 比喻理解：                              │
├─────────────────────────────────────────┤
│ select/poll : 老师点名                  │
│   - 每次问所有学生"谁有作业要交？"       │
│   - 学生举手（事件就绪）→ 老师逐个收     │
│   - 缺点：学生多了点名慢（O(n)）         │
│                                         │
│ epoll    : 课代表汇报                   │
│   - 学生有作业时主动告诉课代表           │
│   - 老师只问课代表"谁要交作业？"         │
│   - 优点：学生再多也高效（O(1)~O(logn)） │
└─────────────────────────────────────────┘
```

### 3.2 select详解

```c
// 📦 函数原型
int select(int nfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval *timeout);

// 🔹 参数解析：
//   nfds      : 监控的最大fd值+1（如监控fd=5,7,10 → nfds=11）
//   readfds   : 关注"可读"事件的fd集合
//   writefds  : 关注"可写"事件的fd集合  
//   exceptfds : 关注"异常"事件的fd集合（通常传NULL）
//   timeout   : 超时控制
//               NULL=永久阻塞
//               {0,0}=非阻塞轮询
//               {5,0}=最多等5秒

// 🔹 返回值：
//   >0 : 就绪的fd总数
//   =0 : 超时，无事件
//   <0 : 错误（errno设置）

// 🔹 fd_set操作宏：
   FD_ZERO(&set);           // 清空集合
   FD_SET(fd, &set);        // 添加fd到集合
   FD_CLR(fd, &set);        // 从集合移除fd
   FD_ISSET(fd, &set);      // 判断fd是否在集合中（就绪）
```

```c
// ✅ select使用示例（单进程多客户端回声）
#define MAX_CLIENTS 10
#define MAX_FD 1024  // ⚠️ select硬限制！

int main() {
    int listen_sock = create_tcp_server(8888);
    int client_fds[MAX_CLIENTS] = {-1};  // 简单连接池
    fd_set readfds, backup_fds;
    
    FD_ZERO(&backup_fds);
    FD_SET(listen_sock, &backup_fds);
    int max_fd = listen_sock;
    
    while (1) {
        readfds = backup_fds;  // ⚠️ select会修改集合，每次需复制
        
        // 阻塞等待事件（超时1秒）
        struct timeval tv = {1, 0};
        int ret = select(max_fd + 1, &readfds, NULL, NULL, &tv);
        if (ret < 0) { perror("select"); break; }
        if (ret == 0) continue;  // 超时，继续循环
        
        // 1. 检查监听socket（新连接）
        if (FD_ISSET(listen_sock, &readfds)) {
            struct sockaddr_in client_addr;
            socklen_t len = sizeof(client_addr);
            int conn_sock = accept(listen_sock, (struct sockaddr*)&client_addr, &len);
            
            // 找空位加入连接池
            for (int i=0; i<MAX_CLIENTS; i++) {
                if (client_fds[i] == -1) {
                    client_fds[i] = conn_sock;
                    FD_SET(conn_sock, &backup_fds);
                    if (conn_sock > max_fd) max_fd = conn_sock;
                    printf("✅ New client fd=%d\n", conn_sock);
                    break;
                }
            }
            FD_CLR(listen_sock, &readfds);  // 已处理，避免重复
        }
        
        // 2. 检查客户端socket（数据接收）
        for (int i=0; i<MAX_CLIENTS; i++) {
            int fd = client_fds[i];
            if (fd < 0 || !FD_ISSET(fd, &readfds)) continue;
            
            char buf[256];
            ssize_t n = recv(fd, buf, sizeof(buf)-1, 0);
            if (n <= 0) {  // 客户端断开或错误
                printf("❌ Client fd=%d disconnected\n", fd);
                close(fd);
                FD_CLR(fd, &backup_fds);
                client_fds[i] = -1;
                continue;
            }
            
            buf[n] = '\0';
            printf("📩 [%d] %s\n", fd, buf);
            send(fd, buf, n, 0);  // 回声
        }
    }
}
```

> ⚠️ **select核心缺陷**（嵌入式慎用原因）：
> ```
> 1️⃣ 文件描述符上限1024（FD_SETSIZE宏定义，修改需重编内核）
> 2️⃣ 每次调用需复制fd_set（用户态↔内核态拷贝，O(n)开销）
> 3️⃣ 返回后需遍历所有fd判断谁就绪（即使只有1个事件）
> 4️⃣ 仅支持水平触发（低速设备），无法边沿触发优化
> ```

### 3.3 poll详解

```c
// 📦 函数原型
int poll(struct pollfd *fds, nfds_t nfds, int timeout);

// 🔹 struct pollfd定义：
   struct pollfd {
       int   fd;      // 监控的文件描述符
       short events;  // 关注的事件（输入）
       short revents; // 实际发生的事件（输出）
   };

// 🔹 事件标志（可|组合）：
   POLLIN   : 可读（数据到达/连接关闭）
   POLLOUT  : 可写（发送缓冲区有空位）
   POLLERR  : 错误（通常自动监控，无需显式设置）
   POLLHUP  : 挂起（对端关闭连接）
   POLLNVAL : 无效fd（编程错误）

// 🔹 返回值：同select（>0就绪数，=0超时，<0错误）
```

```c
// ✅ poll使用示例（改进select的连接池管理）
#define MAX_EVENTS 64

int main() {
    int listen_sock = create_tcp_server(8888);
    struct pollfd fds[MAX_EVENTS];
    int fd_count = 1;  // 当前监控的fd数量
    
    // 初始化：只监控监听socket
    fds[0].fd = listen_sock;
    fds[0].events = POLLIN;
    fds[0].revents = 0;
    
    while (1) {
        int ret = poll(fds, fd_count, 1000);  // 1秒超时
        if (ret < 0) { perror("poll"); break; }
        if (ret == 0) continue;
        
        // 1. 检查监听socket
        if (fds[0].revents & POLLIN) {
            int conn_sock = accept(listen_sock, NULL, NULL);
            if (fd_count < MAX_EVENTS) {
                fds[fd_count].fd = conn_sock;
                fds[fd_count].events = POLLIN;
                fds[fd_count].revents = 0;
                fd_count++;
                printf("✅ New client fd=%d\n", conn_sock);
            } else {
                close(conn_sock);  // 连接池满，拒绝
                fprintf(stderr, "⚠️ Connection pool full\n");
            }
        }
        
        // 2. 检查客户端socket（从后往前遍历，方便删除）
        for (int i = fd_count-1; i > 0; i--) {
            if (fds[i].revents & (POLLIN | POLLHUP | POLLERR)) {
                int fd = fds[i].fd;
                char buf[256];
                ssize_t n = recv(fd, buf, sizeof(buf)-1, 0);
                
                if (n <= 0) {  // 断开或错误
                    printf("❌ Client fd=%d disconnected\n", fd);
                    close(fd);
                    // 用最后一个fd覆盖当前位，避免数组空洞
                    fds[i] = fds[--fd_count];
                } else {
                    buf[n] = '\0';
                    printf("📩 [%d] %s\n", fd, buf);
                    send(fd, buf, n, 0);
                }
            }
        }
    }
}
```

> ⚠️ **poll vs select**：
> ```
> ✅ poll改进：
>    - 无1024限制（链表实现，仅受系统fd上限限制）
>    - 无需每次复制整个集合（只传数组指针）
>    - revents明确返回事件类型，无需FD_ISSET判断
> 
> ❌ poll仍存缺陷：
>    - 仍需遍历所有fd找就绪事件（O(n)）
>    - 仅支持水平触发
>    - 内核仍需扫描整个列表
> ```

### 3.4 epoll详解（⭐嵌入式高并发首选⭐）

```
📦 核心优势：
   1️⃣ 内核维护红黑树+就绪链表 → O(1)~O(logn)事件通知
   2️⃣ 事件表在内核，无需每次用户态↔内核态拷贝
   3️⃣ 返回时直接给出就绪fd列表，无需遍历
   4️⃣ 支持边沿触发（ET）+ 水平触发（LT）双模式
```

#### 🔹 三步操作法

```c
// 步骤1：创建epoll实例
int epoll_create(int size);  // size提示内核预期监控fd数（2.6.8+后仅提示）
// ✅ 返回epoll文件描述符，用于后续操作

// 步骤2：管理监控事件（增/删/改）
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
// 🔹 op参数：
//    EPOLL_CTL_ADD : 添加新fd到监控
//    EPOLL_CTL_MOD : 修改已监控fd的事件
//    EPOLL_CTL_DEL : 从监控中移除fd

// 🔹 struct epoll_event定义：
   typedef union epoll_data {
       void    *ptr;   // 常用：存自定义结构体指针
       int      fd;    // 存关联的fd
       uint32_t u32;
       uint64_t u64;
   } epoll_data_t;

   struct epoll_event {
       uint32_t     events;  // 事件标志（见下文）
       epoll_data_t data;    // 用户数据，事件发生时原样返回
   };

// 🔹 事件标志：
   EPOLLIN      : 可读
   EPOLLOUT     : 可写
   EPOLLERR     : 错误（自动监控）
   EPOLLHUP     : 挂起
   EPOLLET      : ⭐边沿触发模式（默认是水平触发LT）⭐
   EPOLLONESHOT : 一次性触发（事件处理后需重新EPOLL_CTL_MOD）

// 步骤3：等待事件发生
int epoll_wait(int epfd, struct epoll_event *events, 
               int maxevents, int timeout);
// 🔹 返回值：就绪事件数量（直接填充到events数组）
// 🔹 timeout: -1=阻塞，0=非阻塞，>0=毫秒超时
```

#### 🔹 水平触发（LT）vs 边沿触发（ET）

```
📦 核心区别：事件通知的时机

┌─────────────────────────────────────────┐
│ 水平触发（LT, Level-Triggered）- 默认   │
├─────────────────────────────────────────┤
│ 🔹 行为：只要fd处于"就绪状态"，每次epoll_wait都会通知 │
│ 🔹 示例：                              │
│    缓冲区有100字节数据 → 通知可读        │
│    应用只读50字节 → 下次epoll_wait仍通知可读 │
│ 🔹 优点：                              │
│    - 编程简单，不易漏事件               │
│    - 允许分多次读取/写入                │
│ 🔹 缺点：                              │
│    - 可能重复通知，略微增加系统调用      │
│ 🔹 嵌入式建议：⭐默认用LT，简单可靠⭐    │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ 边沿触发（ET, Edge-Triggered）- 需显式设置 │
├─────────────────────────────────────────┤
│ 🔹 行为：仅当fd状态"变化时"通知一次      │
│ 🔹 示例：                              │
│    缓冲区从空→有100字节 → 通知一次可读    │
│    应用只读50字节 → 下次epoll_wait不再通知！│
│ 🔹 优点：                              │
│    - 减少重复通知，理论效率更高          │
│ 🔹 缺点：                              │
│    - 必须一次性读完/写完（循环recv/send）│
│    - 必须配合非阻塞IO + EAGAIN处理       │
│    - 编程复杂，易漏事件导致死锁          │
│ 🔹 嵌入式建议：⚠️ 慎用！仅在高并发+性能瓶颈时考虑 │
└─────────────────────────────────────────┘
```

#### 🔹 epoll完整示例（嵌入式推荐架构）

```c
// ✅ epoll + 非阻塞IO + 连接池（单进程高并发服务器）
#include <sys/epoll.h>
#include <fcntl.h>

#define MAX_EVENTS 64
#define BUF_SIZE 512

// 自定义连接信息（通过epoll_data.ptr传递）
typedef struct {
    int fd;
    char ip[16];
    uint16_t port;
    // 可扩展：状态机、缓冲区、超时时间等
} conn_info_t;

// 设置非阻塞（epoll必须配合非阻塞IO！）
int set_nonblock(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

int main() {
    // 1. 创建监听socket
    int listen_sock = create_tcp_server(8888);
    set_nonblock(listen_sock);  // ✅ 监听socket也建议非阻塞
    
    // 2. 创建epoll实例
    int epfd = epoll_create1(0);  // epoll_create1更安全（无size参数歧义）
    if (epfd < 0) { perror("epoll_create"); return -1; }
    
    // 3. 添加监听socket到epoll（水平触发）
    struct epoll_event ev, events[MAX_EVENTS];
    ev.events = EPOLLIN;  // LT模式（默认）
    ev.data.fd = listen_sock;  // 简单场景直接用data.fd
    epoll_ctl(epfd, EPOLL_CTL_ADD, listen_sock, &ev);
    
    // 4. 事件循环
    while (1) {
        int nready = epoll_wait(epfd, events, MAX_EVENTS, 1000);  // 1秒超时
        if (nready < 0) { 
            if (errno == EINTR) continue;  // 被信号中断，重试
            perror("epoll_wait"); break; 
        }
        if (nready == 0) continue;  // 超时，可执行心跳/清理等任务
        
        for (int i = 0; i < nready; i++) {
            int fd = events[i].data.fd;
            
            // 情况1：新连接请求
            if (fd == listen_sock) {
                struct sockaddr_in client_addr;
                socklen_t len = sizeof(client_addr);
                int conn_sock = accept(listen_sock, (struct sockaddr*)&client_addr, &len);
                if (conn_sock < 0) {
                    if (errno == EAGAIN || errno == EWOULDBLOCK) 
                        break;  // 非阻塞下无更多连接
                    perror("accept"); continue;
                }
                
                set_nonblock(conn_sock);  // ✅ 新连接必须非阻塞
                
                // 分配连接信息（简单场景可用内存池优化）
                conn_info_t *info = malloc(sizeof(conn_info_t));
                info->fd = conn_sock;
                strncpy(info->ip, inet_ntoa(client_addr.sin_addr), 15);
                info->port = ntohs(client_addr.sin_port);
                
                // 添加到epoll监控
                ev.events = EPOLLIN;
                ev.data.ptr = info;  // ✅ 用ptr传递自定义数据
                epoll_ctl(epfd, EPOLL_CTL_ADD, conn_sock, &ev);
                
                printf("✅ New: %s:%d (fd=%d)\n", info->ip, info->port, conn_sock);
            }
            // 情况2：客户端数据/断开
            else {
                conn_info_t *info = (conn_info_t*)events[i].data.ptr;
                int sockfd = info->fd;
                
                if (events[i].events & EPOLLIN) {
                    char buf[BUF_SIZE];
                    ssize_t n;
                    // ✅ ET模式需循环读直到EAGAIN，LT模式可单次读
                    while ((n = recv(sockfd, buf, sizeof(buf)-1, 0)) > 0) {
                        buf[n] = '\0';
                        printf("📩 [%s:%d] %s\n", info->ip, info->port, buf);
                        send(sockfd, buf, n, 0);  // 回声
                    }
                    
                    if (n == 0) {  // 对端正常关闭
                        printf("❌ Closed: %s:%d\n", info->ip, info->port);
                    } else if (errno != EAGAIN && errno != EWOULDBLOCK) {
                        perror("recv");  // 真实错误
                    }
                }
                
                // 处理断开：清理资源
                if (events[i].events & (EPOLLHUP | EPOLLERR) || 
                    (events[i].events & EPOLLIN && /* 上面判断已断开 */ 0)) {
                    epoll_ctl(epfd, EPOLL_CTL_DEL, sockfd, NULL);
                    close(sockfd);
                    free(info);  // ✅ 释放自定义数据
                }
            }
        }
    }
    
    close(listen_sock);
    close(epfd);
    return 0;
}
```

> 💡 **epoll嵌入式最佳实践**：
> ```c
> // ✅ 1. 始终配合非阻塞IO（即使LT模式）
> set_nonblock(fd);  // 避免单次recv阻塞整个事件循环
> 
> // ✅ 2. 用epoll_data.ptr传递上下文（避免全局数组查fd）
> ev.data.ptr = my_conn_context;  // 事件回调时直接获取连接信息
> 
> // ✅ 3. 处理EAGAIN/EWOULDBLOCK（非阻塞IO的正常返回）
> if (n < 0 && (errno == EAGAIN || errno == EWOULDBLOCK)) {
>  // 数据暂缺，下次事件再试 → 正常流程，非错误！
> }
> 
> // ✅ 4. 内存管理：连接建立时malloc，断开时free（或用对象池）
> // ✅ 5. 超时处理：在epoll_wait超时分支执行心跳/清理
> if (nready == 0) {
>  check_connection_timeouts();  // 遍历连接池，清理超时
> }
> 
> // ✅ 6. 优雅退出：捕获SIGTERM/SIGINT，先stop监听+清理连接
> ```

---

## 四、函数接口速查表

### 4.1 fcntl - 文件描述符控制

```c
#include <fcntl.h>
#include <unistd.h>

int fcntl(int fd, int cmd, ... /* arg */);

// 🔹 常用cmd：
//   F_GETFL : 获取fd状态标志 → 返回flags
//   F_SETFL : 设置fd状态标志 → 成功返回0
//   F_SETOWN: 设置接收SIGIO信号的进程ID

// 🔹 常用flags（可|组合）：
   O_NONBLOCK  : 非阻塞模式 ⭐
   O_ASYNC     : 信号驱动IO（慎用）
   O_APPEND    : 追加写入（文件用）

// ✅ 设置非阻塞标准写法：
int set_nonblock(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    if (flags < 0) return -1;
    return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

// ✅ 恢复阻塞模式：
int set_block(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    if (flags < 0) return -1;
    return fcntl(fd, F_SETFL, flags & ~O_NONBLOCK);
}
```

### 4.2 select/poll/epoll 对比速查

| 特性              | select                 | poll                     | epoll（⭐推荐⭐）          |
| ----------------- | ---------------------- | ------------------------ | ------------------------ |
| **最大连接数**    | 1024（FD_SETSIZE）     | 系统fd上限（~10万）      | 系统fd上限（~10万）      |
| **IO效率**        | O(n) 每次遍历所有fd    | O(n) 每次遍历所有fd      | O(1)~O(logn) 仅就绪fd    |
| **内核↔用户拷贝** | 每次调用拷贝整个fd_set | 每次调用拷贝整个数组     | 仅事件发生时拷贝就绪列表 |
| **触发模式**      | 仅水平触发（LT）       | 仅水平触发（LT）         | LT + ET（可选）          |
| **fd复用**        | 每次需重新设置fd_set   | 每次需重新设置events数组 | 一次添加，多次复用       |
| **嵌入式适用性**  | ⚠️ 连接少时可用         | ⚠️ 中等连接数可用         | ✅ 高并发首选             |

---

## 五、嵌入式并发服务器实战架构

### 5.1 推荐架构：epoll + 状态机 + 连接池

```
📦 设计目标：单进程处理100+并发连接，资源可控，无锁竞争

┌─────────────────────────────────────────┐
│ 主循环（epoll_wait）                    │
├─────────────────────────────────────────┤
│ 1. 等待事件（1秒超时）                   │
│ 2. 处理新连接（accept + 初始化状态机）   │
│ 3. 处理就绪连接（根据状态机执行对应操作）│
│ 4. 超时分支：心跳检测 + 清理僵尸连接     │
└─────────────────────────────────────────┘

📦 连接状态机设计（每个连接独立）：
   enum conn_state {
       STATE_HANDSHAKE,  // 握手/认证阶段
       STATE_IDLE,       // 等待指令
       STATE_RECEIVING,  // 接收数据中（处理粘包）
       STATE_PROCESSING, // 业务处理（异步化！）
       STATE_SENDING,    // 发送响应
       STATE_CLOSING     // 优雅关闭
   };

📦 连接上下文结构：
   typedef struct {
       int fd;
       enum conn_state state;
       char recv_buf[1024];  // 接收缓冲区（环形缓冲更佳）
       int recv_pos;         // 已接收字节数
       char send_buf[1024];  // 发送缓冲区
       int send_pos;         // 已发送字节数
       time_t last_active;   // 最后活动时间（超时检测）
       // 业务扩展字段...
   } connection_t;
```

### 5.2 关键代码片段

```c
// ✅ 状态机驱动的数据接收（处理TCP粘包）
void handle_recv(connection_t *conn) {
    ssize_t n = recv(conn->fd, 
                     conn->recv_buf + conn->recv_pos,
                     sizeof(conn->recv_buf) - conn->recv_pos - 1,
                     0);
    
    if (n <= 0) {
        // 断开或错误 → 状态机转移到STATE_CLOSING
        conn->state = STATE_CLOSING;
        return;
    }
    
    conn->recv_pos += n;
    conn->recv_buf[conn->recv_pos] = '\0';
    
    // 尝试解析完整帧（假设以"\r\n"为界）
    char *frame_end = strstr(conn->recv_buf, "\r\n");
    while (frame_end) {
        *frame_end = '\0';  // 截断帧
        process_frame(conn, conn->recv_buf);  // 业务处理
        
        // 移动剩余数据到缓冲区头部
        int frame_len = frame_end - conn->recv_buf + 2;
        memmove(conn->recv_buf, 
                conn->recv_buf + frame_len,
                conn->recv_pos - frame_len);
        conn->recv_pos -= frame_len;
        
        frame_end = strstr(conn->recv_buf, "\r\n");  // 继续解析
    }
    
    // 缓冲区满保护
    if (conn->recv_pos >= sizeof(conn->recv_buf) - 100) {
        fprintf(stderr, "⚠️ Buffer overflow, closing fd=%d\n", conn->fd);
        conn->state = STATE_CLOSING;
    }
}

// ✅ 非阻塞发送（处理EAGAIN）
void handle_send(connection_t *conn) {
    if (conn->send_pos == 0) return;  // 无数据可发
    
    ssize_t n = send(conn->fd,
                     conn->send_buf,
                     conn->send_pos,
                     0);
    
    if (n < 0) {
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            return;  // 发送缓冲区满，下次可写事件再试
        }
        // 真实错误
        conn->state = STATE_CLOSING;
        return;
    }
    
    // 移动未发送数据
    conn->send_pos -= n;
    memmove(conn->send_buf, conn->send_buf + n, conn->send_pos);
    
    if (conn->send_pos == 0) {
        // 发送完成，回到空闲状态
        conn->state = STATE_IDLE;
        // 如果还有待发送数据，可在此处填充conn->send_buf
    }
}

// ✅ epoll事件分发（主循环核心）
void dispatch_event(struct epoll_event *ev) {
    connection_t *conn = (connection_t*)ev->data.ptr;
    
    // 更新最后活动时间
    conn->last_active = time(NULL);
    
    if (ev->events & EPOLLIN) {
        if (conn->state == STATE_RECEIVING || conn->state == STATE_IDLE) {
            conn->state = STATE_RECEIVING;
            handle_recv(conn);
            // 解析出完整帧后，可能触发业务处理→填充send_buf→转STATE_SENDING
        }
    }
    
    if (ev->events & EPOLLOUT) {
        if (conn->state == STATE_SENDING) {
            handle_send(conn);
        }
    }
    
    if (ev->events & (EPOLLHUP | EPOLLERR)) {
        conn->state = STATE_CLOSING;
    }
    
    // 状态机推进：关闭连接
    if (conn->state == STATE_CLOSING) {
        epoll_ctl(epfd, EPOLL_CTL_DEL, conn->fd, NULL);
        close(conn->fd);
        free(conn);  // 或用对象池回收
    }
}
```

### 5.3 资源管理技巧（嵌入式关键！）

```c
// ✅ 1. 连接对象池（避免频繁malloc/free）
#define MAX_CONNECTIONS 128
static connection_t conn_pool[MAX_CONNECTIONS];
static bool conn_used[MAX_CONNECTIONS];

connection_t* alloc_connection() {
    for (int i=0; i<MAX_CONNECTIONS; i++) {
        if (!conn_used[i]) {
            conn_used[i] = true;
            memset(&conn_pool[i], 0, sizeof(connection_t));
            return &conn_pool[i];
        }
    }
    return NULL;  // 池满
}

void free_connection(connection_t *conn) {
    int idx = conn - conn_pool;
    if (idx >=0 && idx < MAX_CONNECTIONS) {
        conn_used[idx] = false;
        // 可选：清空敏感数据
    }
}

// ✅ 2. 超时检测（防止僵尸连接）
void check_timeouts(time_t now) {
    const time_t TIMEOUT_SEC = 300;  // 5分钟无活动
    for (int i=0; i<MAX_CONNECTIONS; i++) {
        if (conn_used[i] && 
            conn_pool[i].state != STATE_CLOSING &&
            now - conn_pool[i].last_active > TIMEOUT_SEC) {
            
            fprintf(stderr, "⏰ Timeout: fd=%d\n", conn_pool[i].fd);
            conn_pool[i].state = STATE_CLOSING;
            // 主循环下次会清理
        }
    }
}

// ✅ 3. 优雅退出（信号处理）
volatile sig_atomic_t g_running = 1;

void sig_handler(int sig) {
    if (sig == SIGTERM || sig == SIGINT) {
        g_running = 0;  // 主循环检测到后退出
    }
}

// 主函数中：
signal(SIGTERM, sig_handler);
signal(SIGINT, sig_handler);

while (g_running) {
    // epoll事件循环...
}
// 退出前清理所有连接
cleanup_all_connections();
```

---

## 六、常见问题与调试技巧

### 6.1 高频问题排查表

| 问题现象                | 可能原因                                           | 解决方案                                  |
| ----------------------- | -------------------------------------------------- | ----------------------------------------- |
| 🔴 epoll返回但无数据可读 | 1. 对端关闭连接（读返回0）2. 非阻塞未处理EAGAIN    | 1. 检查recv返回值==0 2. 判断errno==EAGAIN |
| 🔴 连接无法关闭/资源泄漏 | 1. 未epoll_ctl(DEL) 2. 未close(fd) 3. 未free上下文 | 统一cleanup函数，用goto/RAII风格管理资源  |
| 🔴 高CPU占用             | 1. 忙轮询（非阻塞+无sleep）2. epoll_wait超时设0    | 设置合理超时（100ms~1s），空闲时yield     |
| 🔴 连接数达到上限        | 1. 系统fd限制 2. 连接池满未清理                    | 1. `ulimit -n`调整 2. 实现连接淘汰策略    |
| 🔴 边沿触发漏事件        | 1. 未循环读/写 2. 未处理EAGAIN                     | ET模式必须：循环操作+非阻塞+检查EAGAIN    |
| 🔴 select/poll性能差     | 连接数>100，遍历开销大                             | 迁移到epoll，或优化业务减少连接数         |

### 6.2 调试技巧（嵌入式友好）

```bash
# 🔹 查看进程打开的fd数
ls -l /proc/<pid>/fd | wc -l

# 🔹 查看系统fd上限
ulimit -n  # 用户级
cat /proc/sys/fs/file-max  # 系统级

# 🔹 跟踪系统调用（验证epoll行为）
strace -e epoll_ctl,epoll_wait ./server

# 🔹 压力测试（模拟多客户端）
# 用netcat批量连接：
for i in {1..50}; do 
    echo "msg$i" | nc -q1 192.168.1.100 8888 & 
done

# 🔹 内存泄漏检测（交叉编译valgrind）
valgrind --leak-check=full ./server

# 🔹 嵌入式日志技巧：条件编译+环形日志缓冲区
#ifdef DEBUG
    #define LOG(fmt, ...) fprintf(stderr, "[epoll] " fmt "\n", ##__VA_ARGS__)
#else
    #define LOG(fmt, ...) do {} while(0)
#endif
```

---

## 七、面试高频考点（嵌入式方向）

```
🔥 IO多路复用必问题：
1. Q: select/poll/epoll的本质区别？为什么epoll高效？
   A: select/poll需遍历所有fd（O(n)），epoll用红黑树+就绪链表（O(1)~O(logn)）；
      epoll事件表在内核，避免用户态↔内核态拷贝；返回时直接给就绪列表，无需遍历。

2. Q: 水平触发和边沿触发的区别？嵌入式如何选择？
   A: LT：只要就绪就通知，编程简单；ET：仅状态变化时通知，需循环读写+非阻塞。
      嵌入式默认用LT（可靠优先），仅在高并发+性能瓶颈时考虑ET（需严格测试）。

3. Q: epoll为什么必须配合非阻塞IO？
   A: epoll只通知"就绪"，不保证一次性读完；若用阻塞IO，单次recv可能阻塞整个事件循环。
      非阻塞+EAGAIN处理：数据暂缺时立即返回，下次事件再试。

4. Q: 单进程epoll服务器如何处理阻塞业务逻辑？
   A: ① 业务异步化（状态机+回调）② 线程池：epoll主线程只负责IO，业务丢给工作线程
      ③ 协程（如libco）：在嵌入式资源允许时可考虑。

5. Q: 如何检测客户端异常断开（非正常close）？
   A: ① recv返回0（对端正常关闭）② recv返回-1+errno=ECONNRESET（对端异常）
      ③ epoll事件含EPOLLHUP/EPOLLERR ④ 应用层心跳超时（最可靠）

6. Q: 嵌入式设备实现100+并发连接，资源如何优化？
   A: ① epoll+非阻塞（单进程）② 连接对象池（预分配）③ 精简缓冲区（按业务需求）
      ④ 超时清理（防僵尸）⑤ 禁用调试日志 ⑥ 编译优化（-Os）+ 裁剪无用库
```

---

## 📋 并发模型选型决策树（嵌入式）

```
🎯 需求：设计嵌入式网络服务器
│
├─ 预期并发连接数？
│  ├─ ≤ 5 → ✅ 单循环服务器（简单可靠）
│  └─ > 5 → 进入下一步
│
├─ 业务逻辑是否可能阻塞（>10ms）？
│  ├─ 是 → ✅ epoll主循环 + 工作线程池（2~4线程）
│  └─ 否 → 进入下一步
│
├─ 设备RAM大小？
│  ├─ < 64KB → ✅ select + 固定数组（连接数<10）
│  ├─ 64KB~256KB → ✅ poll + 动态数组（连接数<50）
│  └─ > 256KB → ✅ epoll + 对象池（连接数100+）
│
├─ 是否需要跨进程通信/隔离？
│  ├─ 是 → ⚠️ 多进程（资源允许时）+ 共享内存/Unix Socket
│  └─ 否 → ✅ 保持单进程方案
│
💡 终极建议：
   "嵌入式首选epoll+LT+非阻塞" — 单进程事件驱动架构，资源可控，扩展性好；
   业务复杂时，用"epoll IO线程 + 业务线程池"分离，避免阻塞主循环。
```

---

> ✨ **学习路线建议**：
>
> ```
> 1️⃣ 理解阻塞/非阻塞区别 → 2️⃣ 掌握select基础用法 
> → 3️⃣ 实现poll版多客户端回声 → 4️⃣ 迁移到epoll+状态机 
> → 5️⃣ 添加超时/心跳/对象池 → 6️⃣ 压力测试+性能调优
> ```

> 📌 **核心原则**：  
> **"事件要驱动，状态要清晰；资源要预分配，退出要优雅"**

> 🔗 **延伸学习**：
> - 高性能框架：[libevent](https://libevent.org/) / [libuv](https://libuv.org/)（封装epoll/kqueue）
> - 嵌入式HTTP+epoll：[mongoose](https://github.com/cesanta/mongoose) 源码学习
> - 内核原理：《Linux高性能服务器编程》游双 / 《UNIX网络编程》卷1
> - 实战项目：用epoll实现简易MQTT Broker / Modbus-TCP网关