# 📡 UDP协议与Socket编程

> 📌 适用场景：嵌入式Linux网络编程、物联网通信、实时数据传输

[TOC]

---

## 一、UDP协议核心特性

### 1.1 协议概述
```
📦 UDP: User Datagram Protocol（用户数据报协议）
🎯 定位：传输层无连接、轻量级通信协议

		简单描述：UDP全称用户数据报协议，是无连接的、不可靠但高效的传输层协议。不可靠：不保证数据是否按序/重复送达，可能丢失。适用于：
```

### 1.2 核心特性对比

| 特性                | 详细说明                         | 嵌入式开发影响                  |
| ------------------- | -------------------------------- | ------------------------------- |
| 🔹 **无连接**        | 发送数据前无需三次握手           | ✅ 降低延迟，节省CPU/内存资源    |
| 🔹 **不可靠**        | 不保证送达、不重传、不排序       | ⚠️ 应用层需自行实现可靠性机制    |
| 🔹 **面向报文**      | 保留应用层消息边界，不合并不拆分 | ✅ 适合固定格式指令/传感器数据包 |
| 🔹 **开销小**        | 头部仅8字节，无控制字段          | ✅ 适合带宽受限、低功耗场景      |
| 🔹 **支持广播/组播** | 可向多个目标同时发送             | ✅ 设备发现、固件批量升级        |

### 1.3 典型应用场景
```
🎯 嵌入式首选场景：
├─ 实时视频/音频流（允许少量丢包）
├─ 传感器数据上报（周期发送，丢一帧影响小）
├─ 局域网设备发现（广播/组播）
├─ 控制指令下发（应用层ACK确认）
├─ VNC/远程调试（实时性 > 可靠性）

❌ 不适用场景：
├─ 文件传输（需可靠保证）
├─ 配置参数下载（不能出错）
├─ 关键业务指令（需顺序+确认）
```

---

## 二、网络编程模型

### 2.1 C/S模式（Client/Server）
```c
📐 架构：专用客户端 ↔ 专用服务器
🔄 流程：
    Client                          Server
       │                              │
       │  socket()                    │  socket()
       │  (可选bind())   	 		 │  bind(IP:PORT)
       │                              │  ← 绑定固定端口等待接收
       │  sendto(服务器IP:PORT) ───►   │  recvfrom() 接收数据
       │                              │  处理业务逻辑
       │  ◄─── sendto(客户端) ────     │  （可选回复）
       │  recvfrom() 接收响应          │
       │                              │
       │  close()                     │  close()
```

> 💡 **嵌入式实践**：开发板常作为Client主动上报数据，PC/云服务器作为Server集中接收

### 2.2 B/S模式（Browser/Server）
```
📐 架构：通用浏览器 ↔ Web服务器
🔗 协议：基于HTTP/HTTPS（通常使用TCP）
📌 UDP场景：WebRTC音视频传输底层使用UDP+SRTP
```

---

## 三、Socket编程API详解（UDP）

### 3.1 核心函数速查表

| 函数         | 功能         | 关键参数             | 返回值        | 使用方                    |
| ------------ | ------------ | -------------------- | ------------- | ------------------------- |
| `socket()`   | 创建套接字   | domain/type/protocol | fd(≥0)/-1     | 双方                      |
| `bind()`     | 绑定本地地址 | sockfd/addr/addrlen  | 0/-1          | Server/需固定端口的Client |
| `sendto()`   | 发送数据     | 目标地址+端口        | 发送字节数/-1 | 双方                      |
| `recvfrom()` | 接收数据     | 源地址输出参数       | 接收字节数/-1 | 双方                      |
| `close()`    | 关闭套接字   | sockfd               | 0/-1          | 双方                      |

---

### 3.2 `socket()` - 创建通信端点
```c
#include <sys/socket.h>

int socket(int domain, int type, int protocol);
```

| 参数       | 说明       | UDP常用值                            |
| ---------- | ---------- | ------------------------------------ |
| `domain`   | 协议族     | `AF_INET` (IPv4) / `AF_INET6` (IPv6) |
| `type`     | 套接字类型 | 🔹 `SOCK_DGRAM` (UDP数据报)/          |
| `protocol` | 具体协议   | `0` (自动匹配) / `IPPROTO_UDP`       |

```c
// ✅ 创建UDP套接字示例
int udp_sock = socket(AF_INET, SOCK_DGRAM, 0);
if (udp_sock < 0) {
    perror("socket create failed");
    return -1;
}
```

> ⚠️ **嵌入式注意**：创建后建议设置`SO_REUSEADDR`避免端口占用：
> ```c
> int opt = 1;
> setsockopt(udp_sock, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
> ```

---

### 3.3 地址结构体：`sockaddr` vs `sockaddr_in`

```c
// 🔹 通用地址结构（函数参数要求）
struct sockaddr {
    sa_family_t sa_family;      // 地址族：AF_INET
    char        sa_data[14];    // 地址数据（14字节）
};

// 🔹 IPv4专用结构（编程时使用，最后强转为sockaddr*）
struct sockaddr_in {
    sa_family_t    sin_family;  // 地址族：AF_INET
    in_port_t      sin_port;    // 端口号（网络字节序！）
    struct in_addr sin_addr;    // IP地址（网络字节序）
    char           sin_zero[8]; // 填充字节，置0
};

// 🔹 IP地址结构
struct in_addr {
    uint32_t s_addr;            // 32位IP（网络字节序）
};
```

#### 🔸 地址初始化模板
```c
struct sockaddr_in server_addr;
memset(&server_addr, 0, sizeof(server_addr));  // 先清零！

server_addr.sin_family = AF_INET;              // IPv4
server_addr.sin_port   = htons(8888);          // 端口转网络字节序
server_addr.sin_addr.s_addr = inet_addr("192.168.1.100");  // 字符串IP转二进制
// 或绑定本机所有IP：
// server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
```

---

### 3.4 字节序转换函数（❗关键❗）
```
🌐 主机字节序：小端（Little-Endian，x86/ARM常用）
🌐 网络字节序：大端（Big-Endian，协议标准）
🔥 所有端口号、IP地址在网络传输前必须转换！
```

| 函数      |            | 功能                         | 示例                                |
| --------- | ---------- | ---------------------------- | ----------------------------------- |
| `htonl()` | 主机转网络 | Host to Network Long (32位)  | `ip = htonl(0xC0A80164);`           |
| `htons()` | 主转网     | Host to Network Short (16位) | `port = htons(8888);`               |
| `ntohs()` | 网转主     | Network to Host Short        | `port = ntohs(addr.sin_port);`      |
| `ntohl()` | 网转主     | Network to Host Long         | `ip = ntohl(addr.sin_addr.s_addr);` |

```
📝 记忆口诀：h=host, n=network, s=short(16bit), l=long(32bit)
```

---

### 3.5 IP地址转换函数

```c
#include <arpa/inet.h>

// 🔹 字符串IP → 32位二进制（网络字节序）
in_addr_t inet_addr(const char *cp);
// 示例：
uint32_t ip_bin = inet_addr("192.168.1.100");  
// 返回：0xC0A80164 (网络字节序)

// 🔹 32位二进制 → 字符串IP（注意：返回静态缓冲区，非线程安全）
char *inet_ntoa(struct in_addr in);
// 示例：
struct in_addr ip;
ip.s_addr = htonl(0xC0A80164);
printf("IP: %s\n", inet_ntoa(ip));  // 输出：192.168.1.100

// 🔹 推荐：线程安全版本（新系统支持）
int inet_pton(int af, const char *src, void *dst);  // 字符串→二进制
const char *inet_ntop(int af, const void *src, char *dst, socklen_t size);  // 二进制→字符串
```

---

### 3.6 `bind()` - 绑定本地地址（服务端必需）
```c
#include <sys/socket.h>

int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

| 参数      | 说明                              |
| --------- | --------------------------------- |
| `sockfd`  | socket()返回的文件描述符          |
| `addr`    | `(struct sockaddr*)&本地地址结构` |
| `addrlen` | `sizeof(struct sockaddr_in)`      |

```c
// ✅ Server端绑定示例
struct sockaddr_in local_addr;
local_addr.sin_family = AF_INET;
local_addr.sin_port = htons(8888);
local_addr.sin_addr.s_addr = htonl(INADDR_ANY);  // 监听所有网卡

if (bind(udp_sock, (struct sockaddr*)&local_addr, sizeof(local_addr)) < 0) {
    perror("bind failed");
    close(udp_sock);
    return -1;
}
printf("✅ Server bound to 0.0.0.0:8888\n");
```

> 💡 **Client是否需要bind?**  
> - 不bind：系统自动分配临时端口（49152~65535）  
> - bind：固定源端口（用于防火墙策略/协议要求）

---

### 3.7 `sendto()` - 发送数据（无连接）
```c
#include <sys/socket.h>

ssize_t sendto(int sockfd, const void *buf, size_t len, int flags,
               const struct sockaddr *dest_addr, socklen_t addrlen);
```

| 参数        | 说明                                             |
| ----------- | ------------------------------------------------ |
| `sockfd`    | 套接字描述符                                     |
| `buf`       | 发送数据缓冲区                                   |
| `len`       | 数据长度                                         |
| `flags`     | 发送标志（通常传0）                              |
| `dest_addr` | 🔥 目标地址结构 `(struct sockaddr*)&dest_addr_in` |
| `addrlen`   | `sizeof(struct sockaddr_in)`                     |

```c
// ✅ 发送数据示例
struct sockaddr_in dest;
dest.sin_family = AF_INET;
dest.sin_port = htons(9999);
dest.sin_addr.s_addr = inet_addr("192.168.1.200");

char msg[] = "Hello UDP";
ssize_t sent = sendto(udp_sock, msg, strlen(msg), 0, 
                      (struct sockaddr*)&dest, sizeof(dest));
if (sent < 0) {
    perror("sendto failed");
} else {
    printf("✅ Sent %zd bytes\n", sent);
}
```

> ⚠️ **UDP发送注意**：
> - 不保证对方收到，无重传机制
> - 单次发送建议 ≤ 1472字节（避免IP分片：1500-20-8）
> - 嵌入式传感器数据建议打包成固定结构体+校验和

---

### 3.8 `recvfrom()` - 接收数据（阻塞/非阻塞）
```c
#include <sys/socket.h>

ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags,
                 struct sockaddr *src_addr, socklen_t *addrlen);
```

| 参数       | 说明                                         |
| ---------- | -------------------------------------------- |
| `src_addr` | 🔥 输出参数：存放发送方地址（可传NULL）       |
| `addrlen`  | 🔥 输入输出参数：传入sizeof，返回实际地址长度 |

```c
// ✅ 接收数据示例（带发送方信息）
struct sockaddr_in client_addr;
socklen_t client_len = sizeof(client_addr);
char recv_buf[1500];

ssize_t n = recvfrom(udp_sock, recv_buf, sizeof(recv_buf)-1, 0,
                     (struct sockaddr*)&client_addr, &client_len);
if (n > 0) {
    recv_buf[n] = '\0';  // 添加字符串结束符
    printf("📩 From %s:%d >> %s\n", 
           inet_ntoa(client_addr.sin_addr),
           ntohs(client_addr.sin_port),
           recv_buf);
    
    // 🔁 可选：原路回复
    // sendto(udp_sock, "ACK", 3, 0, (struct sockaddr*)&client_addr, client_len);
}
```

> 🔥 **阻塞特性**：默认阻塞等待数据，嵌入式建议：
> ```c
> // 设置为非阻塞 + select/poll 多路复用
> int flags = fcntl(udp_sock, F_GETFL, 0);
> fcntl(udp_sock, F_SETFL, flags | O_NONBLOCK);
> ```

---

## 四、UDP报文头结构



```
📦 UDP头部固定8字节，格式如下：

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          源端口(16)          |        目的端口(16)            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          长度(16)            |         校验和(16)             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         数据 (可变)                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| 字段         | 长度 | 说明                                                |
| ------------ | ---- | --------------------------------------------------- |
| **源端口**   | 16位 | 发送方进程端口，可为0（不关心回复时）               |
| **目的端口** | 16位 | 接收方进程端口，必须指定                            |
| **长度**     | 16位 | UDP(头部+数据)总字节数，UDP头为8字节，所以最小值=8B |
| **校验和**   | 16位 | 伪头部+UDP头+数据校验，❌ 嵌入式常设为0（节省计算）  |

> 💡 **伪头部校验**：校验和计算时临时添加12字节伪头部（源IP+目的IP+协议+长度），仅用于校验，不传输

---

## 五、嵌入式UDP编程完整示例

### 5.1 简易UDP Server（接收+回复）
```c
// udp_server.c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define PORT 8888
#define BUF_SIZE 1024

int main() {
    // 1. 创建UDP套接字
    int sock = socket(AF_INET, SOCK_DGRAM, 0);
    if (sock < 0) { perror("socket"); return -1; }

    // 2. 绑定地址
    struct sockaddr_in server_addr = {0};
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    server_addr.sin_port = htons(PORT);
    
    if (bind(sock, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        perror("bind"); close(sock); return -1;
    }
    printf("✅ UDP Server running on port %d\n", PORT);

    // 3. 循环接收+回复
    char buf[BUF_SIZE];
    struct sockaddr_in client_addr;
    socklen_t client_len = sizeof(client_addr);
    
    while (1) {
        ssize_t n = recvfrom(sock, buf, BUF_SIZE-1, 0,
                            (struct sockaddr*)&client_addr, &client_len);
        if (n > 0) {
            buf[n] = '\0';
            printf("📩 [%s:%d] %s\n", 
                   inet_ntoa(client_addr.sin_addr),
                   ntohs(client_addr.sin_port), buf);
            
            // 原路回复
            char reply[] = "ACK received";
            sendto(sock, reply, strlen(reply), 0,
                  (struct sockaddr*)&client_addr, client_len);
        }
    }
    close(sock);
    return 0;
}
```

### 5.2 简易UDP Client（发送+接收）
```c
// udp_client.c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define SERVER_IP "192.168.1.100"
#define SERVER_PORT 8888
#define BUF_SIZE 1024

int main() {
    // 1. 创建套接字（Client可不bind）
    int sock = socket(AF_INET, SOCK_DGRAM, 0);
    if (sock < 0) { perror("socket"); return -1; }

    // 2. 配置服务器地址
    struct sockaddr_in server_addr = {0};
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERVER_PORT);
    server_addr.sin_addr.s_addr = inet_addr(SERVER_IP);

    // 3. 循环发送
    char buf[BUF_SIZE];
    while (1) {
        printf("📤 Input message: ");
        fgets(buf, BUF_SIZE, stdin);
        buf[strcspn(buf, "\n")] = 0;  // 去除换行符
        
        if (sendto(sock, buf, strlen(buf), 0,
                  (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
            perror("sendto"); break;
        }
        
        // 等待回复（可选）
        char reply[BUF_SIZE];
        struct sockaddr_in from;
        socklen_t from_len = sizeof(from);
        ssize_t n = recvfrom(sock, reply, BUF_SIZE-1, 0,
                            (struct sockaddr*)&from, &from_len);
        if (n > 0) {
            reply[n] = '\0';
            printf("✅ Reply: %s\n", reply);
        }
    }
    close(sock);
    return 0;
}
```

### 5.3 编译与测试
```bash
# 交叉编译（嵌入式）
arm-linux-gnueabihf-gcc udp_server.c -o udp_server
arm-linux-gnueabihf-gcc udp_client.c -o udp_client

# 本机测试
gcc udp_server.c -o server && ./server
gcc udp_client.c -o client && ./client

# 测试命令
./client
📤 Input message: hello embedded
📩 [192.168.1.100:8888] hello embedded
✅ Reply: ACK received
```

---

## 六、Wireshark抓包调试指南

### 6.1 安装与启动
```bash
# Ubuntu/Debian
sudo apt update
sudo apt-get install wireshark -y
# 添加当前用户到wireshark组（避免每次sudo）
sudo usermod -aG wireshark $USER
# 重新登录生效

# 启动（图形界面）
sudo wireshark
# 或命令行抓包
sudo tshark -i any -f "udp port 8888" -w capture.pcap
```

### 6.2 关键过滤表达式
```
🔹 基础过滤：
  udp.port == 8888          # 指定端口
  ip.addr == 192.168.1.100  # 指定IP
  udp                        # 仅显示UDP

🔹 组合过滤：
  (udp.port == 8888) && (ip.src == 192.168.1.200)
  udp && frame.len < 100    # 小数据包分析

🔹 协议字段过滤：
  udp.srcport == 5000
  udp.payload contains "hello"  # 搜索数据内容
```

### 6.3 UDP包深度分析要点
```
🔍 选中UDP包 → 展开"User Datagram Protocol"：
├─ Source Port: 54321          ← 客户端临时端口
├─ Destination Port: 8888      ← 服务端监听端口
├─ Length: 20                  ← 8头+12数据
├─ Checksum: 0x1a2b [validated] ← 校验和（0表示未计算）
└─ [Payload]: 68656c6c6f...   ← 十六进制数据，右键→"追迹流"看ASCII
```

> 💡 **嵌入式调试技巧**：
> ```bash
> # 开发板抓包（无GUI）：
> tcpdump -i eth0 -nn -X udp port 8888 -w /tmp/udp.pcap
> # 传输到PC用Wireshark分析：
> scp root@192.168.1.100:/tmp/udp.pcap ./
> wireshark udp.pcap
> ```

---

## 七、作业与实践拓展

### 🔹 作业1：UDP文件传输（简化版）
```
📁 需求：
  ./udp_server 8888 output.bin    # 服务端：监听8888，保存数据到文件
  ./udp_client 192.168.1.100 8888 input.bin  # 客户端：发送文件

⚠️ 关键设计：
1. 分块发送：每次≤1400字节，避免分片
2. 简单校验：每包添加序列号+简易checksum
3. 应用层ACK：接收方回复"ACK:seq"，发送方超时重传
4. 结束标志：最后一包带EOF标记

📦 数据包格式建议：
  [4B seq][4B total][4B offset][2B len][N data][2B checksum]
```

### 🔹 作业2：UDP全双工聊天室
```
💬 需求：
  - 两人同时运行程序，可互相发送消息
  - 显示对方IP:端口 + 消息内容 + 时间戳
  - 支持"/quit"退出

🚀 进阶挑战：
  ✅ 非阻塞IO + select实现同时收发
  ✅ 多客户端支持（服务端维护客户端列表）
  ✅ 消息加密（简易XOR或AES-128）
  ✅ 断线检测（心跳包机制）

💡 核心代码片段（非阻塞+select）：
  fd_set readfds;
  FD_ZERO(&readfds);
  FD_SET(STDIN_FILENO, &readfds);   // 监听键盘
  FD_SET(udp_sock, &readfds);        // 监听网络
  
  if (select(udp_sock+1, &readfds, NULL, NULL, &tv) > 0) {
      if (FD_ISSET(STDIN_FILENO, &readfds)) {
          // 读取键盘输入并sendto
      }
      if (FD_ISSET(udp_sock, &readfds)) {
          // recvfrom接收并显示
      }
  }
```

---

## 八、嵌入式UDP开发避坑指南

### ⚠️ 常见问题与解决方案

| 问题现象       | 可能原因                                         | 解决方案                                                     |
| -------------- | ------------------------------------------------ | ------------------------------------------------------------ |
| 🔴 收不到数据   | 1. 防火墙拦截 2. 端口未bind 3. IP/端口字节序错误 | 1. `sudo ufw disable`测试 2. 确认server已bind 3. 检查`htons`/`inet_addr` |
| 🔴 数据乱码     | 1. 未加`'\0'` 2. 字节序未转换 3. 结构体填充字节  | 1. 接收后手动`buf[n]='\0'` 2. 多字节字段`ntohl` 3. `__attribute__((packed))` |
| 🔴 程序阻塞     | `recvfrom`默认阻塞                               | 设置`O_NONBLOCK` + `select`/`poll`                           |
| 🔴 大数据丢失   | 超过MTU导致分片，分片丢失整个包                  | 控制单次`sendto` ≤ 1400字节                                  |
| 🔴 多客户端冲突 | 服务端未记录客户端地址                           | 用`recvfrom`获取`src_addr`，回复时用该地址                   |

### ✅ 嵌入式最佳实践
```c
// 1. 套接字健壮性配置
int opt = 1;
setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));  // 端口复用
setsockopt(sock, SOL_SOCKET, SO_BROADCAST, &opt, sizeof(opt));  // 允许广播（如需）

// 2. 接收缓冲区调整（避免丢包）
int rcvbuf = 256*1024;  // 256KB
setsockopt(sock, SOL_SOCKET, SO_RCVBUF, &rcvbuf, sizeof(rcvbuf));

// 3. 超时设置（避免永久阻塞）
struct timeval timeout = {2, 0};  // 2秒
setsockopt(sock, SOL_SOCKET, SO_RCVTIMEO, &timeout, sizeof(timeout));

// 4. 错误处理模板
#define UDP_CHECK(expr, msg) do { \
    if ((expr) < 0) { \
        perror(msg); \
        goto cleanup; \
    } \
} while(0)
```

---

## 九、面试高频考点

```
🔥 UDP必问题：
1. Q: UDP如何保证可靠性？
   A: 应用层实现：①序列号+确认机制 ②超时重传 ③校验和 ④流量控制（如MQTT-SN）

2. Q: UDP最大传输单元是多少？
   A: 理论65507字节(65535-8头-20IP头)，但建议≤1472字节避免以太网分片(1500-20-8)

3. Q: select/poll/epoll在UDP编程中的区别？
   A: 
      - select: 跨平台，1024连接限制，线性扫描
      - poll: 无连接数限制，仍线性扫描
      - epoll: Linux高效，O(1)事件通知

4. Q: 如何检测对方是否离线？
   A: UDP无连接状态，需应用层心跳：①定期发PING ②超时未回复判定离线 ③指数退避重连

5. Q: 嵌入式设备用UDP传传感器数据，如何设计协议？
   A: 参考Modbus-UDP：[设备ID][功能码][数据长度][数据][CRC16]，固定帧头+校验+超时重传
```

---

> ✨ **学习路线建议**：
> ```
> 1️⃣ 理解UDP特性 → 2️⃣ 掌握Socket API → 3️⃣ 实现Echo程序 
> → 4️⃣ 添加可靠性机制 → 5️⃣ 结合MQTT/自定义协议 → 6️⃣ 压力测试+抓包分析
> ```

> 📌 **核心原则**：  
> **"网络不可靠，代码要健壮；资源要节省，设计要精简"**
