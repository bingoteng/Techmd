# 进程间通信（`IPC`）

[TOC]

------

## 1.什么是进程间通信？

- 进程间通信是指**不同进程之间进行数据交换或信息传递**的机制
- 由于进程间地址空间独立，需要**内核提供的特殊机制**实现通信
- 通信目的：🟢 *数据传输、资源共享、通知事件、进程控制*

---

## 2.为什么需要进程间通信？

- 进程间内存隔离：每个进程有独立的虚拟地址空间，无法直接访问对方内存 💡 *这是操作系统稳定性和安全性的基础*
- 协作需求：多个进程需要协同完成复杂任务
- 数据共享：需要在进程间传递数据或共享资源
- 控制需求：一个进程需要控制或通知另一个进程

🟢 *典型应用场景*：

```
✅ Shell管道：ls -l | grep .txt
✅ 客户端-服务器：Web浏览器与Web服务器
✅ 生产者-消费者：数据采集进程与处理进程
✅ 微服务架构：不同服务进程间的通信
```

---

## 3.IPC方式分类

### 按通信方式分类：

1. **管道（Pipe）**
   - 匿名管道：亲缘关系进程间
   - 命名管道（FIFO）：任意进程间

2. **信号（Signal）**
   - 异步通知机制

3. **消息队列（Message Queue）**
   - 内核维护的消息链表

4. ✅**共享内存（Shared Memory）**
   - 最快的`IPC`方式

5. **信号量（Semaphore）**
   - 主要用于同步

6. ✅**套接字（Socket）**
   - 不同主机间通信

### 按通信特性对比：

| `IPC`方式 | 数据传输    | 同步机制     | 适用场景   | 效率  |
| --------- | ----------- | ------------ | ---------- | ----- |
| 匿名管道  | ✅           | 阻塞IO       | 亲缘进程   | 中    |
| 命名管道  | ✅           | 阻塞IO       | 任意进程   | 中    |
| 信号      | ❌（仅通知） | 异步         | 事件通知   | 高    |
| 消息队列  | ✅           | 可选         | 任意进程   | 中    |
| 共享内存  | ✅           | 需配合信号量 | 大数据传输 | 🔥最高 |
| 信号量    | ❌（计数）   | 同步         | 进程同步   | 高    |
| 套接字    | ✅           | 可选         | 网络通信   | 低-中 |

💡 *面试高频：共享内存最快，因为数据不需要在用户态和内核态之间拷贝*

---

## 4.管道（Pipe）

### 1.匿名管道

**特点**：

- 只能用于**具有亲缘关系**的进程间（父子、兄弟）
- **半双工**通信（数据单向流动）
- 必须先创建管道再fork
- 存在于内存中，生命周期随进程
- 基于字节流（无消息边界）

```c
man 2 pipe
#include <unistd.h>

int pipe(int pipefd[2]);

功能：创建匿名管道
参数：
    @pipefd[2] --- 文件描述符数组
        pipefd[0]: 读端
        pipefd[1]: 写端
返回值：
    成功 返回 0
    失败 返回 -1，设置errno
```

**使用框架**：

```c
int fd[2];
pipe(fd);  // 创建管道

pid_t pid = fork();

if (pid > 0) {
    // 父进程：写
    close(fd[0]);  // 关闭读端
    write(fd[1], "hello", 6);
    close(fd[1]);
    wait(NULL);
} else if (pid == 0) {
    // 子进程：读
    close(fd[1]);  // 关闭写端
    char buf[100];
    read(fd[0], buf, sizeof(buf));
    close(fd[0]);
}
```

🟡 **注意事项**：

```c
✅ 读写前关闭不用的端口（避免阻塞）
✅ 管道满时 写 阻塞（默认64KB）
✅ 管道空时 读 阻塞
✅ 所有写端关闭后，读read返回0（EOF）
✅ 所有读端关闭后，写write触发SIGPIPE信号，管道破裂

💡从输入/输出角度看
写为输入-读为输出
输入>输出，管道写满了，写端阻塞
输入<输出，管道读完了，读端阻塞
无读想写-管道破裂
写满写阻塞，读空读阻塞；无读就写，管道破裂！
```

💡 **管道容量**：

```
Linux默认管道缓冲区大小：64KB（4.10+内核）
查看：❗ulimit -a | grep pipe
修改：/proc/sys/fs/pipe-buffer-size
```

### 2.命名管道（FIFO）

**特点**：

- 任意进程间通信（通过文件名标识）
- 在文件系统中存在（特殊文件）
- **半双工**通信
- 遵循FIFO原则

```c
man 3 mkfifo
#include <sys/types.h>
#include <sys/stat.h>

int mkfifo(const char *pathname, mode_t mode);

功能：创建命名管道
参数：
    @pathname --- 管道文件路径
    @mode --- 权限（如0666）
返回值：
    成功 返回 0
    失败 返回 -1
```

**使用示例**：

```c
// 进程A（写）
mkfifo("/tmp/myfifo", 0666);
int fd = open("/tmp/myfifo", O_WRONLY);
write(fd, "hello", 6);
close(fd);

// 进程B（读）
int fd = open("/tmp/myfifo", O_RDONLY);
char buf[100];
read(fd, buf, sizeof(buf));
close(fd);
unlink("/tmp/myfifo");  // 删除管道文件
```

🟢 **命名管道 vs 匿名管道**：

```
| 特性 | 匿名管道 | 命名管道 |
|------|----------|----------|
| 创建方式 | pipe() | mkfifo() |
| 存在形式 | 内存 | 文件系统 |
| 适用进程 | 亲缘关系 | 任意进程 |
| 生命周期 | 进程结束即销毁 | 需unlink删除 |
| 查看 | 不可见 | ls -l可见（p开头） |
```

---

## 5.信号（Signal）

### 1.信号基础

**概念**：

- 软件中断机制
- 异步通知进程发生了某个事件
- 信号是**进程间通信最简单的形式**（仅传递信号编号）

💡 **常见信号**：

```
SIGINT (2)   : Ctrl+C 中断
SIGQUIT (3)  : Ctrl+\ 退出
SIGKILL (9)  : 强制终止（不可捕获）
SIGSEGV (11) : 段错误 
SIGPIPE (13) : 管道破裂
SIGALRM (14) : 定时器超时
SIGTERM (15) : 优雅终止（默认）
SIGCHLD (17) : 子进程状态改变
SIGCONT (18) : 继续
SIGSTOP (19) : 暂停
```

### 2.信号发送

```c
man 2 kill
#include <sys/types.h>
#include <signal.h>

int kill(pid_t pid, int sig);

功能：发送信号给进程或进程组
参数：
    @pid:
        >0  : 发送给指定PID的进程
        =0  : 发送给同进程组的所有进程
        =-1 : 发送给所有有权限的进程
        <-1 : 发送给指定进程组
    @sig: 信号编号（0用于检测进程是否存在）
返回值：
    成功 返回 0
    失败 返回 -1
```

```c
man 2 alarm
#include <unistd.h>

unsigned int alarm(unsigned int seconds);

功能：设置定时器，seconds秒后发送SIGALRM信号
参数：@seconds --- 秒数（0取消定时器）
返回值：之前定时器剩余的秒数
```

### 3.信号处理

- 忽略
- 默认
- 捕捉：9号-19号信号无法忽略捕捉

```c
man 2 signal
#include <signal.h>

typedef void (*sighandler_t)(int);
sighandler_t signal(int signum, sighandler_t handler);

功能：设置信号处理函数
参数：
    @signum --- 信号编号
    @handler:
        SIG_IGN   : 忽略信号
        SIG_DFL   : 默认处理
        函数指针  : 自定义处理函数
返回值：之前的处理函数地址，失败返回SIG_ERR
```

**示例**：

```c
#include <stdio.h>
#include <signal.h>
#include <unistd.h>

void sig_handler(int sig) {
    printf("收到信号: %d\n", sig);
}

int main() {
    signal(SIGINT, sig_handler);  // 捕获Ctrl+C
    
    while(1) {
        printf("运行中...\n");
        sleep(1);
    }
    return 0;
}
```

🟡 **signal vs sigaction**：

```
⚠️ signal()在不同系统行为不一致（可移植性差）
✅ 推荐使用sigaction()（POSIX标准）

int sigaction(int signum, const struct sigaction *act,
              struct sigaction *oldact);
```

### 4.信号特点

🔴 **信号局限性**：

```
❌ 不能传递大量数据（仅信号编号）
❌ 信号可能丢失（相同信号不排队）
❌ 异步性导致竞态条件
❌ 信号处理函数中只能调用异步信号安全函数
```

💡 **异步信号安全函数**（可在信号处理函数中调用）：

```
✅ _exit(), signal(), kill()
✅ read(), write()（仅限描述符操作）
✅ 大部分算术运算
❌ printf(), malloc(), exit()（不安全！）
```

---

## 6.消息队列（Message Queue）

### 1.概念

- **内核空间维护的消息链表、独立于两个进程空间**
- 消息有类型和格式
- 克服了信号信息量小的限制
- 克服了管道无消息边界的限制

  ```c
  命令
  ipcs
  查看IPC对象的信息
  ipcs -m/q/s 
  m：共享内存
  q：消息队列
  s：信号灯
  
  ipcrm 
  删除IPC对象
  ipcrm -m/q/s  shmid/msgid/semid 
  ipcrm -M/-Q/-S  key
  ```

### 2.消息队列操作

#### 1.创建消息队列

```c
man 2 msgget
#include <sys/msg.h>

int msgget(key_t key, int msgflg);

功能：创建或获取消息队列
参数：
    @key: 
        IPC_PRIVATE : 创建私有队列（亲缘进程）
        ftok()返回值 : 公共队列（任意进程）
    @msgflg:
        IPC_CREAT : 不存在则创建
        IPC_EXCL  : 与IPC_CREAT一起，存在则失败
        权限位    : 如0666
返回值：
    成功 返回消息队列ID（非负整数）
    失败 返回 -1
```

#### 2.生成IPC对象名称

```c
man 3 ftok 
key_t ftok(const char *pathname, int proj_id);
功能：
    利用pathname和proj_id生成一个key_t类型的key值（IPC对象名称）
参数：
    pathname:路径
    proj_id:项目ID
返回值：
    成功返回key_t类型的key值
    失败返回-1 
```

#### 3.向消息队列中发送消息

```c
man 2 msgsnd
#include <sys/msg.h>

int msgsnd(int msqid, const void *msgp, size_t msgsz, int msgflg);

功能：发送消息到队列
参数：
    @msqid --- 消息队列ID
    @msgp --- 消息结构体指针
    @msgsz --- 消息正文大小（不含类型）
    @msgflg:
        0      : 队列满时阻塞
        IPC_NOWAIT : 队列满时立即返回
返回值：
    成功 返回 0
    失败 返回 -1
```

#### 4.从消息队列中接收消息

```c
man 2 msgrcv
#include <sys/msg.h>

ssize_t msgrcv(int msqid, void *msgp, size_t msgsz, long msgtyp, int msgflg);

功能：从队列接收消息
参数：
    @msqid --- 消息队列ID
    @msgp --- 接收缓冲区指针
    @msgsz --- 缓冲区大小
    @msgtyp:
        0      : 接收第一条消息
        >0     : 接收第一条类型为msgtyp的消息
        <0     : 接收第一条类型小于|msgtyp|的消息
    @msgflg:
        0      : 无消息时阻塞
        IPC_NOWAIT : 无消息时立即返回
返回值：
    成功 返回实际接收的字节数
    失败 返回 -1
```

#### 5.控制消息队列

```c
man 2 msgctl
#include <sys/msg.h>

int msgctl(int msqid, int cmd, struct msqid_ds *buf);

功能：消息队列控制
参数：
    @msqid --- 消息队列ID
    @cmd:
        IPC_STAT : 获取队列属性
        IPC_SET  : 设置队列属性
        IPC_RMID : 删除队列
    @buf --- 属性结构体指针
返回值：
    成功 返回 0
    失败 返回 -1
```

### 3.消息结构体

```c
// 消息结构体（第一个成员必须是long类型）
struct msgbuf {
    long mtype;       // 消息类型（必须>0）
    char mtext[100];  // 消息正文
};

// 或使用指针
struct mymsg {
    long type;
    void *data;
    size_t len;
};
```

### 4.使用示例

```c
// 进程A（发送）
#include <sys/msg.h>
#include <stdio.h>
#include <string.h>

struct msgbuf {
    long mtype;
    char mtext[100];
};

int main() {
    int msgid = msgget(IPC_PRIVATE, 0666 | IPC_CREAT);
    
    if (fork() > 0) {
        // 父进程：发送
        struct msgbuf msg;
        msg.mtype = 1;
        strcpy(msg.mtext, "Hello from parent");
        msgsnd(msgid, &msg, strlen(msg.mtext), 0);
        wait(NULL);
        msgctl(msgid, IPC_RMID, NULL);
    } else {
        // 子进程：接收
        struct msgbuf msg;
        msgrcv(msgid, &msg, sizeof(msg.mtext), 1, 0);
        printf("收到: %s\n", msg.mtext);
    }
    return 0;
}
```

🟢 **消息队列优缺点**：

```
✅ 优点：
   - 消息有格式，可区分优先级（mtype）
   - 独立于进程，队列不随进程结束
   - 可实现随机查询式通信

❌ 缺点：
   - 消息拷贝到内核空间，效率不如共享内存
   - 消息长度有限制
   - 需要手动管理队列生命周期
```

💡 **查看消息队列**：

```bash
ipcs -q          # 查看消息队列
ipcrm -q <id>    # 删除消息队列
```

---

## 7.共享内存（Shared Memory）

### 1.概念

- **最快的`IPC方式`** 💡 *面试必考*
- ✅**多个进程映射同一块物理内存**
- 数据**无需在用户态和内核态之间拷贝**
- 需要配合同步机制（信号量）防止竞态

### 2.共享内存操作

```c
man 2 shmget
#include <sys/ipc.h>
#include <sys/shm.h>

int shmget(key_t key, size_t size, int shmflg);

功能：创建或获取共享内存段
参数：
    @key:
        IPC_PRIVATE : 私有共享内存
        ftok()返回值 : 公共共享内存
    @size --- 共享内存大小（字节）
    @shmflg:
        IPC_CREAT : 不存在则创建
        IPC_EXCL  : 与IPC_CREAT一起，存在则失败
        权限位    : 如0666
返回值：
    成功 返回共享内存ID（非负整数）
    失败 返回 -1
```

```c
man 2 shmat
#include <sys/shm.h>

void *shmat(int shmid, const void *shmaddr, int shmflg);

功能：将共享内存附加到进程地址空间
参数：
    @shmid --- 共享内存ID
    @shmaddr:
        NULL  : 由系统选择地址（推荐）
        非NULL: 指定地址（需shmflg包含SHM_RND）
    @shmflg:
        0      : 可读可写
        SHM_RDONLY : 只读
返回值：
    成功 返回共享内存起始地址
    失败 返回 (void*)-1
```

```c
man 2 shmdt
#include <sys/shm.h>

int shmdt(const void *shmaddr);

功能：将共享内存从进程地址空间分离
参数：@shmaddr --- shmat返回的地址
返回值：
    成功 返回 0
    失败 返回 -1
```

```c
man 2 shmctl
#include <sys/shm.h>

int shmctl(int shmid, int cmd, struct shmid_ds *buf);

功能：共享内存控制
参数：
    @shmid --- 共享内存ID
    @cmd:
        IPC_STAT : 获取属性
        IPC_SET  : 设置属性
        IPC_RMID : 标记删除（最后一个进程分离后真正删除）
    @buf --- 属性结构体指针
返回值：
    成功 返回 0
    失败 返回 -1
```

### 3.使用示例

```c
// 进程A（创建+写入）
#include <sys/ipc.h>
#include <sys/shm.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

int main() {
    // 创建共享内存
    int shmid = shmget(IPC_PRIVATE, 1024, 0666 | IPC_CREAT);
    
    // 附加到地址空间
    char *shm = (char*)shmat(shmid, NULL, 0);
    
    if (fork() > 0) {
        // 父进程：写
        strcpy(shm, "Hello from parent");
        sleep(2);  // 等待子进程读取
        printf("父进程收到: %s\n", shm);
        
        // 分离和删除
        shmdt(shm);
        shmctl(shmid, IPC_RMID, NULL);
        wait(NULL);
    } else {
        // 子进程：读
        sleep(1);
        printf("子进程收到: %s\n", shm);
        strcpy(shm, "Hello from child");
        shmdt(shm);
    }
    return 0;
}
```

### 4.共享内存+信号量（完整同步）

```c
// 生产者-消费者模型
#include <sys/ipc.h>
#include <sys/shm.h>
#include <semaphore.h>
#include <stdio.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>

#define SHM_SIZE 1024

typedef struct {
    sem_t empty;  // 空槽位信号量
    sem_t full;   // 满槽位信号量
    sem_t mutex;  // 互斥锁
    char buffer[SHM_SIZE];
} shared_data;

int main() {
    int shmid = shmget(IPC_PRIVATE, sizeof(shared_data), 0666 | IPC_CREAT);
    shared_data *data = (shared_data*)shmat(shmid, NULL, 0);
    
    // 初始化信号量
    sem_init(&data->empty, 1, 1);  // pshared=1表示进程间共享
    sem_init(&data->full, 1, 0);
    sem_init(&data->mutex, 1, 1);
    
    if (fork() > 0) {
        // 生产者
        sem_wait(&data->empty);
        sem_wait(&data->mutex);
        strcpy(data->buffer, "Hello");
        sem_post(&data->mutex);
        sem_post(&data->full);
        
        wait(NULL);
        shmdt(data);
        shmctl(shmid, IPC_RMID, NULL);
    } else {
        // 消费者
        sem_wait(&data->full);
        sem_wait(&data->mutex);
        printf("收到: %s\n", data->buffer);
        sem_post(&data->mutex);
        sem_post(&data->empty);
        
        shmdt(data);
    }
    return 0;
}
```

🟢 **共享内存优缺点**：

```
✅ 优点：
   - 🚀 速度最快（零拷贝）
   - 适合大数据传输
   - 灵活（可定义任意数据结构）

❌ 缺点：
   - 需要额外的同步机制（信号量）
   - 编程复杂度高
   - 需要手动管理生命周期
```

💡 **查看共享内存**：

```bash
ipcs -m          # 查看共享内存
ipcrm -m <id>    # 删除共享内存
```

---

## 8.信号量集/信号灯（Semaphore）

### 1.概念

- 主要用于**进程同步**（也可用于互斥）
- 计数器，表示可用资源数量
- 配合共享内存使用

🟡 **注意**：线程间使用无名信号量（`sem_init`），进程间使用有名信号量（`sem_open`）或共享内存中的无名信号量

### 2.有名信号量操作

```c
man 3 sem_open
#include <semaphore.h>
#include <fcntl.h>

sem_t *sem_open(const char *name, int oflag);
sem_t *sem_open(const char *name, int oflag, mode_t mode, unsigned int value);

功能：打开或创建有名信号量
参数：
    @name --- 信号量名称（以/开头，如"/mysem"）
    @oflag:
        O_CREAT : 不存在则创建
        O_EXCL  : 与O_CREAT一起，存在则失败
    @mode --- 权限（如0666）
    @value --- 初始值（仅创建时需要）
返回值：
    成功 返回信号量指针
    失败 返回 SEM_FAILED
```

```c
man 3 sem_wait
#include <semaphore.h>

int sem_wait(sem_t *sem);  // P操作
int sem_post(sem_t *sem);  // V操作
int sem_trywait(sem_t *sem);  // 非阻塞P操作
int sem_timedwait(sem_t *sem, const struct timespec *abs_timeout);

功能：信号量PV操作
参数：@sem --- 信号量指针
返回值：
    成功 返回 0
    失败 返回 -1
```

```c
man 3 sem_close
#include <semaphore.h>

int sem_close(sem_t *sem);

功能：关闭信号量（进程内）
参数：@sem --- 信号量指针
返回值：成功 0，失败 -1
```

```c
man 3 sem_unlink
#include <semaphore.h>

int sem_unlink(const char *name);

功能：删除信号量（系统范围）
参数：@name --- 信号量名称
返回值：成功 0，失败 -1
```

### 3.使用示例

```c
// 进程A
sem_t *sem = sem_open("/mysem", O_CREAT, 0666, 1);
sem_wait(sem);  // P操作
// 临界区
sem_post(sem);  // V操作
sem_close(sem);
sem_unlink("/mysem");

// 进程B
sem_t *sem = sem_open("/mysem", 0);  // 仅打开
sem_wait(sem);
// 临界区
sem_post(sem);
sem_close(sem);
```

💡 **有名 vs 无名信号量**：

```
| 特性 | 有名信号量 | 无名信号量 |
|------|------------|------------|
| 创建 | sem_open() | sem_init() |
| 标识 | 名称 | 内存地址 |
| 存在 | 文件系统 | 内存 |
| 适用 | 进程间 | 线程间/共享内存中的进程间 |
| 删除 | sem_unlink() | sem_destroy() |
```

---

## 9.套接字（Socket）

### 1.概念

- 最通用的IPC方式
- 可用于**同一主机**或**不同主机**间通信
- 支持多种协议（TCP、UDP、UNIX Domain Socket）

### 2.UNIX Domain Socket（本地套接字）

**特点**：

- 同一主机内进程通信
- 比网络套接字快（不走网络协议栈）
- 支持流式（SOCK_STREAM）和数据报（SOCK_DGRAM）

```c
#include <sys/socket.h>
#include <sys/un.h>
#include <unistd.h>

// 创建
int socket(int domain, int type, int protocol);
// domain = AF_UNIX (本地)

// 绑定
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);

// 监听（服务器）
int listen(int sockfd, int backlog);

// 接受连接（服务器）
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);

// 连接（客户端）
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);

// 读写
ssize_t send(int sockfd, const void *buf, size_t len, int flags);
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
```

🟢 **Socket vs 其他IPC**：

```
✅ 优点：
   - 通用性强（本地/网络都可用）
   - 支持一对多通信
   - 成熟的API和工具

❌ 缺点：
   - 性能不如共享内存
   - 编程复杂度高
   - 需要处理连接管理
```

---

## 10.IPC方式选择指南

### 决策树：

```
1. 是否需要跨主机通信？
   ├── 是 → Socket（TCP/UDP）
   └── 否 → 继续

2. 进程是否有亲缘关系？
   ├── 是 → 匿名管道
   └── 否 → 继续

3. 传输数据量大小？
   ├── 小（<4KB） → 消息队列/管道
   ├── 中（4KB-64KB） → 消息队列/命名管道
   └── 大（>64KB） → 共享内存

4. 是否需要实时性？
   ├── 是 → 共享内存+信号量
   └── 否 → 消息队列

5. 仅需要通知，不需要数据？
   └── 信号
```

### 对比总结表：

| IPC方式  | 数据量 | 速度  | 复杂度 | 适用场景         |
| -------- | ------ | ----- | ------ | ---------------- |
| 信号     | 无     | 最快  | 低     | 事件通知         |
| 匿名管道 | 中     | 中    | 低     | 亲缘进程流式数据 |
| 命名管道 | 中     | 中    | 中     | 任意进程流式数据 |
| 消息队列 | 小-中  | 中    | 中     | 结构化消息       |
| 共享内存 | 大     | 🚀最快 | 高     | 大数据、高性能   |
| 信号量   | 无     | 快    | 中     | 同步控制         |
| Socket   | 任意   | 低-中 | 高     | 网络/本地通用    |

💡 **面试高频总结**：

```
1. 最快的IPC：共享内存（零拷贝）
2. 最简单的IPC：信号（仅通知）
3. 最通用的IPC：Socket（本地/网络）
4. 管道 vs 消息队列：管道无消息边界，队列有
5. 共享内存必须配合同步机制（信号量）
```

---

## 🎯 整体优化建议

1. **代码规范**：所有示例添加错误检查（返回值判断、errno处理）
2. **资源清理**：强调IPC资源的清理（msgctl/shmctl/sem_unlink）
3. **调试工具**：🟢 *补充ipcs/ipcrm命令使用*
4. **实战案例**：添加完整的生产者-消费者模型示例
5. **嵌入式关联**：🟢 *结合你的背景，可补充：*
   - 嵌入式Linux中IPC资源限制配置
   - 多线程+多进程混合架构设计
   - 实时系统中的IPC选择（优先级继承等）

> ✨ **一句话总结IPC核心**：  
> *"根据数据量、实时性、进程关系选择合适的IPC方式；共享内存最快但需同步，管道最简单但仅限亲缘，Socket最通用但开销大"*

这份笔记涵盖了Linux主要IPC机制，配合之前的进程和线程笔记，可形成完整的并发编程知识体系！🚀