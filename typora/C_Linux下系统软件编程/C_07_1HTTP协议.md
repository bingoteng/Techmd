# 🌐 HTTP协议与Socket编程

> 📌 适用场景：嵌入式物联网数据上报、RESTful API调用、Web配置界面、OTA升级

[TOC]

---

## 一、HTTP协议核心特性

### 1.1 协议概述

```
📦 HTTP: HyperText Transfer Protocol（超文本传输协议）
🎯 定位：应用层协议，基于TCP的可靠数据传输，万维网数据通信基础
📐 所属模型：
   ┌─────────────────┐
   │  应用层：HTTP    │ ← 本笔记重点
   ├─────────────────┤
   │  传输层：TCP     │ ← 提供可靠连接
   ├─────────────────┤
   │  网络层：IP      │ ← 提供路由寻址
   ├─────────────────┤
   │  网络接口层       │ ← 物理传输
   └─────────────────┘
```

### 1.2 核心特性

| 特性               | HTTP详细说明                        | 嵌入式开发影响                  |
| ------------------ | ----------------------------------- | ------------------------------- |
| 🔹 **应用层协议**   | 定义请求/响应语义，人类可读         | ✅ 调试方便，协议标准化          |
| 🔹 **无状态**       | 每次请求独立，服务器不保存上下文    | ⚠️ 需Cookie/Token实现会话管理    |
| 🔹 **灵活扩展**     | 头部字段可自定义，支持多种内容类型  | ✅ 适配JSON/XML/二进制等多种格式 |
| 🔹 **基于TCP**      | 依赖TCP可靠传输，默认端口80/443     | ⚠️ 继承TCP的延迟和开销           |
| 🔹 **文本协议**     | 报文头部为ASCII文本，便于解析       | ✅ 嵌入式可用字符串函数解析      |
| 🔹 **支持持久连接** | Keep-Alive复用TCP连接，减少握手开销 | ✅ 高频请求场景必备              |

### 1.3 典型应用场景

```
🎯 嵌入式首选场景：
├─ IoT设备数据上报（JSON+HTTP POST到云平台）
├─ 固件OTA升级（HTTP GET下载固件包）
├─ Web配置界面（嵌入式HTTP Server + HTML）
├─ RESTful API调用（获取天气/时间/认证服务）
├─ 日志远程上传（HTTP POST + gzip压缩）

❌ 不适用场景：
├─ 实时控制指令（延迟敏感，用TCP/UDP自定义协议）
├─ 高频小数据包（HTTP头部开销大，用MQTT/CoAP）
├─ 极低功耗设备（HTTP解析消耗CPU/内存）
├─ 局域网设备发现（用UDP广播+自定义协议）
```

---

## 二、万维网核心三要素

### 2.1 URL：统一资源定位符

```
📍 格式：
<协议>://<主机>[:<端口>]/<路径>[?<查询参数>][#片段]

📦 示例解析：
   https://api.weather.com:443/v1/forecast?city=beijing&unit=c#tomorrow
   │      │              │   │           │              │
   │      │              │   │           │              └─ 片段（客户端用）
   │      │              │   │           └─ 查询参数（键值对）
   │      │              │   └─ 资源路径
   │      │              └─ 端口（默认80/443可省略）
   │      └─ 主机（域名或IP）
   └─ 协议（http/https）
```

```c
// ✅ 嵌入式URL解析示例（简化版）
typedef struct {
    char scheme[8];    // "http"/"https"
    char host[64];     // 域名或IP
    uint16_t port;     // 端口
    char path[128];    // 路径
    char query[256];   // 查询参数
} url_t;

int parse_url(const char *url_str, url_t *url) {
    // 1. 解析协议
    if (sscanf(url_str, "%7[^:]://%63[^:/]%*s", 
               url->scheme, url->host) != 2) return -1;
    
    // 2. 解析端口（默认80/443）
    url->port = (strcmp(url->scheme, "https") == 0) ? 443 : 80;
    char *port_str = strstr(url_str, ":");
    if (port_str && *(port_str+1) != '/') {
        sscanf(port_str+1, "%hu", &url->port);
    }
    
    // 3. 解析路径和查询
    char *path_start = strstr(url_str, "/");
    if (path_start) {
        char *query_start = strchr(path_start, '?');
        if (query_start) {
            strncpy(url->path, path_start, query_start - path_start);
            strncpy(url->query, query_start+1, sizeof(url->query)-1);
        } else {
            strncpy(url->path, path_start, sizeof(url->path)-1);
        }
    }
    return 0;
}
```

### 2.2 HTTP：超文本传输协议

```
🔄 工作过程（基于TCP）：
    Client                          Server
       │                              │
       │  socket()+connect()          │  socket()+bind()+listen()
       │  [TCP三次握手]                │  accept() → conn_sock
       │                              │
       │  send() HTTP请求报文 ─────►   │  
       │                              │  recv() 解析请求行+头部
       │                              │  业务逻辑处理
       │  ◄──── send() HTTP响应 ───   │  
       │                              │
       │  recv() 解析响应+正文          │  
       │  [可选：Keep-Alive复用连接]    │  
       │                              │
       │  close() [TCP四次挥手]        │  close()
```

### 2.3 HTML：超文本标记语言

```
📄 作用：定义网页结构和内容，浏览器渲染显示
🔗 与HTTP关系：HTTP传输HTML文档，HTML中可引用其他资源（CSS/JS/图片）

📦 嵌入式简化场景：
   - 设备配置页面：轻量级HTML + 表单提交
   - 状态监控页：定时刷新 + JSON数据嵌入
   - OTA升级页：文件上传表单 + 进度显示

💡 嵌入式建议：
   - 使用微型HTTP服务器：mongoose / civetweb / lwip-httpd
   - HTML精简：移除CSS/JS，用内联样式
   - 动态内容：用sprintf拼接变量到HTML模板
```

---

## 三、HTTP报文格式详解

### 3.1 请求报文结构

```
📦 格式：
<请求行>\r\n
<请求头部>: <值>\r\n
<请求头部>: <值>\r\n
...
\r\n                   ← 空行（头部结束标志）
[请求正文]              ← 可选，POST/PUT时携带

📋 请求行格式：
<方法> <请求目标> <HTTP版本>
GET /api/weather?city=beijing HTTP/1.1\r\n
```

#### 🔹 请求方法对比

| 方法        | 含义                       | 嵌入式典型场景          | 是否带正文 |
| ----------- | -------------------------- | ----------------------- | ---------- |
| **GET**     | 获取资源                   | ✅ 查询天气/配置/状态    | ❌ 无       |
| **POST**    | 提交数据，创建资源         | ✅ 数据上报/表单提交     | ✅ 有       |
| **PUT**     | 更新资源（幂等）           | ⚠️ 配置参数更新          | ✅ 有       |
| **DELETE**  | 删除资源                   | ⚠️ 删除设备/日志         | ❌ 无       |
| **HEAD**    | 只获取响应头（不返回正文） | ✅ 检查资源是否存在/大小 | ❌ 无       |
| **OPTIONS** | 查询服务器支持的方法       | ⚠️ CORS预检请求          | ❌ 无       |

> 💡 **嵌入式建议**：90%场景只需实现`GET`+`POST`，减少代码复杂度

### 3.2 响应报文结构

```
📦 格式：
<状态行>\r\n
<响应头部>: <值>\r\n
<响应头部>: <值>\r\n
...
\r\n                   ← 空行（头部结束标志）
[响应正文]              ← 可选

📋 状态行格式：
<HTTP版本> <状态码> <原因短语>
HTTP/1.1 200 OK\r\n
```

#### 🔹 常用状态码速查

| 分类       | 状态码 | 含义             | 嵌入式处理建议            |
| ---------- | ------ | ---------------- | ------------------------- |
| **2xx**    | 200    | ✅ 请求成功       | 解析正文，执行业务逻辑    |
| 成功       | 201    | ✅ 资源创建成功   | 记录新资源ID/路径         |
| **3xx**    | 301    | 🔁 永久重定向     | 更新本地缓存的URL         |
| 重定向     | 302    | 🔁 临时重定向     | 临时跳转，不更新缓存      |
| **4xx**    | 400    | ❌ 请求参数错误   | 检查URL/头部/正文格式     |
| 客户端错误 | 401    | ❌ 未授权         | 携带Token/证书重试        |
|            | 403    | ❌ 禁止访问       | 检查权限配置              |
|            | 404    | ❌ 资源不存在     | 检查URL路径，记录错误日志 |
| **5xx**    | 500    | 💥 服务器内部错误 | 重试+指数退避，上报监控   |
| 服务端错误 | 502    | 💥 网关错误       | 检查代理/负载均衡配置     |
|            | 503    | 💥 服务不可用     | 延迟重试，降级处理        |

### 3.3 关键头部字段

```
🔹 请求头部常用字段：
   Host: api.example.com          ← 必需，虚拟主机识别
   User-Agent: Embedded/1.0       ← 客户端标识，便于服务端统计
   Accept: application/json       ← 声明期望的响应格式
   Content-Type: application/json ← POST时声明正文格式
   Content-Length: 128            ← 正文长度（字节）
   Connection: keep-alive         ← 持久连接，复用TCP
   Authorization: Bearer <token>  ← 认证令牌

🔹 响应头部常用字段：
   Content-Type: application/json; charset=utf-8  ← 正文MIME类型+编码
   Content-Length: 256            ← 正文长度（便于解析）
   Transfer-Encoding: chunked     ← 分块传输（不定长正文）
   Set-Cookie: session=abc123     ← 服务器设置Cookie
   Cache-Control: no-cache        ← 缓存策略
   Server: nginx/1.18             ← 服务器软件信息（可隐藏）
```

> ⚠️ **嵌入式注意**：
> - `Content-Length` 和 `Transfer-Encoding: chunked` 二选一，不能同时出现
> - 解析响应时**必须先读完头部**，再根据`Content-Length`或`chunked`规则读正文

---

## 四、HTTP/1.1 vs HTTP/2 vs HTTPS

### 4.1 版本对比

| 特性           | HTTP/1.1（主流）       | HTTP/2（进阶）              | HTTPS（安全）                |
| -------------- | ---------------------- | --------------------------- | ---------------------------- |
| **传输方式**   | 文本协议，头部冗余     | 二进制分帧，头部压缩(HPACK) | HTTP + TLS/SSL加密           |
| **多路复用**   | ❌ 串行请求（队头阻塞） | ✅ 同连接并发多请求          | ✅ 同HTTP/2 + 加密            |
| **头部压缩**   | ❌ 每次重复发送         | ✅ HPACK算法，大幅减少开销   | ✅ 同HTTP/2                   |
| **服务器推送** | ❌ 需客户端请求         | ✅ 服务器主动推送关联资源    | ✅ 同HTTP/2                   |
| **嵌入式支持** | ✅ 易实现，库丰富       | ⚠️ 复杂，资源消耗大          | ⚠️ 需TLS库（mbedTLS/wolfSSL） |
| **默认端口**   | 80                     | 80（升级协商）              | 443                          |

### 4.2 嵌入式选型建议

```
🎯 资源受限设备（<256KB RAM）：
   ✅ HTTP/1.1 + 短连接 + 精简头部
   ❌ 避免HTTP/2（解析复杂）和HTTPS（TLS开销大）

🎯 中等资源设备（256KB~1MB RAM）：
   ✅ HTTP/1.1 + Keep-Alive + 基础TLS（可选）
   ⚠️ 可尝试轻量级HTTP/2实现（如nghttp2嵌入式移植）

🎯 高安全要求场景（支付/医疗）：
   ✅ 必须HTTPS + 证书验证 + TLS 1.2+
   🔧 使用硬件加速（如ESP32的SSL加速）

💡 实用技巧：
   - 使用`Accept-Encoding: gzip` + 服务端压缩，减少传输量
   - 自定义User-Agent便于服务端识别设备类型
   - 超时设置：连接3s + 读取5s + 写入3s（根据网络质量调整）
```

---

## 五、嵌入式HTTP Client编程示例

### 5.1 简易HTTP GET请求（获取天气数据）

```c
// http_client.c - 嵌入式HTTP GET示例
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define SERVER "api.weather.com"
#define PORT 80
#define BUF_SIZE 2048

// 发送原始HTTP请求
int http_get(const char *host, const char *path, char *response, int resp_size) {
    // 1. 创建TCP套接字
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) { perror("socket"); return -1; }
    
    // 2. 连接服务器（简化：假设已解析IP）
    struct sockaddr_in server = {0};
    server.sin_family = AF_INET;
    server.sin_port = htons(PORT);
    server.sin_addr.s_addr = inet_addr("1.2.3.4");  // 实际需用gethostbyname解析域名
    
    if (connect(sock, (struct sockaddr*)&server, sizeof(server)) < 0) {
        perror("connect"); close(sock); return -1;
    }
    
    // 3. 构造HTTP请求（注意：\r\n换行，空行结束头部）
    char request[512];
    snprintf(request, sizeof(request),
             "GET %s HTTP/1.1\r\n"
             "Host: %s\r\n"
             "User-Agent: Embedded/1.0\r\n"
             "Accept: application/json\r\n"
             "Connection: close\r\n"  // 短连接，简化处理
             "\r\n", path, host);
    
    // 4. 发送请求
    if (send(sock, request, strlen(request), 0) < 0) {
        perror("send"); close(sock); return -1;
    }
    
    // 5. 接收响应（简化：一次性读完）
    int total = 0;
    while (1) {
        int n = recv(sock, response + total, resp_size - total - 1, 0);
        if (n <= 0) break;  // 错误或关闭
        total += n;
        if (total >= resp_size - 1) break;  // 缓冲区满
    }
    response[total] = '\0';
    
    close(sock);
    return total;  // 返回接收字节数
}

// 解析响应：分离头部和正文
int parse_http_response(const char *resp, char **body_start, int *body_len) {
    // 查找空行"\r\n\r\n"（头部结束标志）
    char *header_end = strstr(resp, "\r\n\r\n");
    if (!header_end) return -1;
    
    *body_start = header_end + 4;  // 跳过空行
    *body_len = strlen(*body_start);
    
    // 可选：解析Content-Length验证长度
    char *cl = strstr(resp, "Content-Length:");
    if (cl) {
        int expected_len = atoi(cl + 15);  // 跳过"Content-Length:"
        if (expected_len != *body_len) {
            fprintf(stderr, "⚠️ Content-Length mismatch: %d vs %d\n", 
                    expected_len, *body_len);
        }
    }
    return 0;
}

int main() {
    char response[4096] = {0};
    
    // 发送GET请求
    int recv_len = http_get(SERVER, "/v1/now?city=beijing&key=xxx", 
                           response, sizeof(response));
    if (recv_len < 0) {
        fprintf(stderr, "❌ HTTP request failed\n");
        return -1;
    }
    
    // 解析响应
    char *body;
    int body_len;
    if (parse_http_response(response, &body, &body_len) < 0) {
        fprintf(stderr, "❌ Parse response failed\n");
        return -1;
    }
    
    // 输出状态行和正文（简化）
    printf("📦 Response (%d bytes):\n%.*s\n", body_len, body_len, body);
    
    // ✅ 实际场景：用cJSON解析JSON正文
    // cJSON *json = cJSON_Parse(body);
    // if (json) { /* 提取温度/湿度等字段 */ cJSON_Delete(json); }
    
    return 0;
}
```

### 5.2 HTTP POST请求（JSON数据上报）

```c
// 发送JSON数据到云平台
int http_post_json(const char *host, const char *path, 
                   const char *json_data, char *response, int resp_size) {
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    // ... [连接代码同上，省略] ...
    
    // 构造POST请求（注意Content-Length和Content-Type）
    char request[1024];
    snprintf(request, sizeof(request),
             "POST %s HTTP/1.1\r\n"
             "Host: %s\r\n"
             "Content-Type: application/json\r\n"
             "Content-Length: %zu\r\n"  // 🔥 关键：必须准确
             "Connection: close\r\n"
             "\r\n"
             "%s",  // 🔥 注意：空行后直接跟JSON正文
             path, host, strlen(json_data), json_data);
    
    // 发送 + 接收逻辑同上...
    // ...
}

// 调用示例
int main() {
    // 构造JSON数据（实际建议用cJSON库）
    char json[256];
    snprintf(json, sizeof(json),
             "{\"device_id\":\"DEV001\",\"temp\":25.6,\"hum\":60}");
    
    char response[2048];
    http_post_json("iot.cloud.com", "/api/v1/upload", 
                   json, response, sizeof(response));
    
    // 解析响应：{"code":200,"msg":"ok"}
    // ...
}
```

### 5.3 域名解析（gethostbyname）

```c
// 嵌入式DNS解析（简化版，实际需处理错误和IPv6）
#include <netdb.h>

int resolve_host(const char *hostname, char *ip_str, int ip_size) {
    struct hostent *he = gethostbyname(hostname);
    if (!he || he->h_addrtype != AF_INET) return -1;
    
    struct in_addr **addr_list = (struct in_addr **)he->h_addr_list;
    strncpy(ip_str, inet_ntoa(*addr_list[0]), ip_size - 1);
    return 0;
}

// 使用示例
char ip[16];
if (resolve_host("api.weather.com", ip, sizeof(ip)) == 0) {
    printf("✅ Resolved: %s\n", ip);  // 输出: 1.2.3.4
    // 用ip替换connect()中的inet_addr()参数
}
```

> ⚠️ **嵌入式注意**：
> - `gethostbyname` 是阻塞调用，建议设置DNS超时（`/etc/resolv.conf`）
> - 资源极受限设备：预配置服务器IP，避免DNS解析
> - 推荐使用`getaddrinfo`（支持IPv6，但代码稍复杂）

---

## 六、JSON数据解析（嵌入式轻量方案）

### 6.1 JSON格式速查

```
📦 基础语法：
{
  "key1": "string_value",      // 字符串
  "key2": 123,                 // 数字
  "key3": true,                // 布尔
  "key4": null,                // 空值
  "key5": ["a", "b", "c"],    // 数组
  "key6": {"nested": "obj"}    // 嵌套对象
}

📦 嵌入式典型数据：
{
  "code": 200,
  "data": {
    "city": "beijing",
    "weather": "sunny",
    "temp": 25.6,
    "update_time": "2024-03-29 14:30"
  }
}
```

### 6.2 轻量解析方案对比

| 方案                  | 优点               | 缺点                 | 适用场景           |
| --------------------- | ------------------ | -------------------- | ------------------ |
| **手动strstr+sscanf** | 零依赖，代码可控   | 易出错，不支持嵌套   | 极简场景，固定格式 |
| **cJSON库** ⭐推荐     | 成熟稳定，功能完整 | ~20KB代码+内存开销   | 通用嵌入式项目     |
| **json-c库**          | 功能强大，标准兼容 | ~50KB，较复杂        | 资源较丰富设备     |
| **自定义微型解析器**  | 按需裁剪，极致轻量 | 开发成本高，易漏边界 | 资源<64KB的MCU     |

### 6.3 cJSON使用示例（推荐）

```c
// 编译：gcc http_client.c -lcjson -o client
#include "cJSON.h"

// 解析天气JSON响应
int parse_weather_json(const char *json_str, float *temp, char *weather) {
    cJSON *root = cJSON_Parse(json_str);
    if (!root) return -1;
    
    // 检查状态码
    cJSON *code = cJSON_GetObjectItem(root, "code");
    if (!cJSON_IsNumber(code) || code->valueint != 200) {
        cJSON_Delete(root);
        return -1;
    }
    
    // 提取嵌套数据
    cJSON *data = cJSON_GetObjectItem(root, "data");
    if (!cJSON_IsObject(data)) {
        cJSON_Delete(root);
        return -1;
    }
    
    // 获取温度（数字）
    cJSON *temp_item = cJSON_GetObjectItem(data, "temp");
    if (cJSON_IsNumber(temp_item)) {
        *temp = (float)temp_item->valuedouble;
    }
    
    // 获取天气（字符串）
    cJSON *weather_item = cJSON_GetObjectItem(data, "weather");
    if (cJSON_IsString(weather_item)) {
        strncpy(weather, weather_item->valuestring, 32);
    }
    
    cJSON_Delete(root);  // 🔥 必须释放内存
    return 0;
}

// 调用
float temp;
char weather[32];
if (parse_weather_json(body, &temp, weather) == 0) {
    printf("🌤️ %s, %.1f°C\n", weather, temp);
}
```

> 💡 **嵌入式优化**：
> ```c
> // 1. 静态分配cJSON对象池（避免频繁malloc）
> // 2. 使用cJSON_Minify()预处理，移除空白字符节省内存
> // 3. 只解析必需字段，跳过无关数据
> // 4. 编译时裁剪：#define cJSON_StringsAsStrings  // 避免strdup
> ```

---

## 七、嵌入式HTTP Server简易实现

### 7.1 最小HTTP Server（响应静态页面）

```c
// mini_http_server.c - 单线程，单连接，仅支持GET
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define PORT 8080
#define WEB_ROOT "/www"  // 网页根目录

// 发送HTTP响应
void send_response(int client_sock, int status_code, 
                   const char *content_type, const char *body) {
    const char *status_text = (status_code == 200) ? "OK" : 
                              (status_code == 404) ? "Not Found" : "Error";
    
    char header[256];
    snprintf(header, sizeof(header),
             "HTTP/1.1 %d %s\r\n"
             "Content-Type: %s\r\n"
             "Content-Length: %zu\r\n"
             "Connection: close\r\n"
             "\r\n",
             status_code, status_text, content_type, strlen(body));
    
    send(client_sock, header, strlen(header), 0);
    if (body) send(client_sock, body, strlen(body), 0);
}

// 处理单个请求
void handle_request(int client_sock) {
    char request[1024] = {0};
    recv(client_sock, request, sizeof(request)-1, 0);
    
    // 解析请求行：GET /index.html HTTP/1.1
    char method[16], path[128], version[16];
    if (sscanf(request, "%15s %127s %15s", method, path, version) != 3) {
        send_response(client_sock, 400, "text/plain", "Bad Request");
        return;
    }
    
    // 仅支持GET
    if (strcmp(method, "GET") != 0) {
        send_response(client_sock, 405, "text/plain", "Method Not Allowed");
        return;
    }
    
    // 安全处理路径：防止目录遍历
    if (strstr(path, "..") || path[0] != '/') {
        send_response(client_sock, 403, "text/plain", "Forbidden");
        return;
    }
    
    // 构建文件路径
    char file_path[256];
    snprintf(file_path, sizeof(file_path), "%s%s", WEB_ROOT, 
             strcmp(path, "/") == 0 ? "/index.html" : path);
    
    // 读取文件内容
    FILE *f = fopen(file_path, "r");
    if (!f) {
        send_response(client_sock, 404, "text/plain", "404 Not Found");
        return;
    }
    
    // 获取文件内容（简化：小文件一次性读取）
    fseek(f, 0, SEEK_END);
    long fsize = ftell(f);
    fseek(f, 0, SEEK_SET);
    
    char *content = malloc(fsize + 1);
    fread(content, 1, fsize, f);
    content[fsize] = '\0';
    fclose(f);
    
    // 确定Content-Type（简化版）
    const char *ext = strrchr(file_path, '.');
    const char *ctype = "application/octet-stream";
    if (ext) {
        if (strcmp(ext, ".html") == 0) ctype = "text/html";
        else if (strcmp(ext, ".css") == 0) ctype = "text/css";
        else if (strcmp(ext, ".js") == 0) ctype = "application/javascript";
        else if (strcmp(ext, ".json") == 0) ctype = "application/json";
    }
    
    send_response(client_sock, 200, ctype, content);
    free(content);
}

int main() {
    int server_sock = socket(AF_INET, SOCK_STREAM, 0);
    
    // 地址复用 + 绑定
    int opt = 1;
    setsockopt(server_sock, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    
    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    addr.sin_port = htons(PORT);
    
    bind(server_sock, (struct sockaddr*)&addr, sizeof(addr));
    listen(server_sock, 3);
    
    printf("🌐 HTTP Server running on port %d, web_root=%s\n", PORT, WEB_ROOT);
    
    // 单连接循环（实际应用需用select/epoll支持并发）
    while (1) {
        struct sockaddr_in client_addr;
        socklen_t client_len = sizeof(client_addr);
        int client_sock = accept(server_sock, (struct sockaddr*)&client_addr, &client_len);
        if (client_sock < 0) continue;
        
        printf("🔗 Connection from %s\n", inet_ntoa(client_addr.sin_addr));
        handle_request(client_sock);
        close(client_sock);
    }
    
    close(server_sock);
    return 0;
}
```

### 7.2 动态内容：表单处理示例

```c
// 处理GET查询参数：/search?keyword=esp32
void handle_search(int client_sock, const char *query) {
    // 解析query字符串：keyword=esp32&limit=10
    char key[32], value[64];
    if (sscanf(query, "keyword=%63s", value) == 1) {
        // 生成动态HTML响应
        char response[512];
        snprintf(response, sizeof(response),
                 "<!DOCTYPE html><html><body>"
                 "<h1>Search: %s</h1>"
                 "<p>Results: ESP32 development board, ESP32-CAM...</p>"
                 "</body></html>", value);
        send_response(client_sock, 200, "text/html", response);
    } else {
        send_response(client_sock, 400, "text/plain", "Missing keyword");
    }
}

// 处理POST表单提交（Content-Type: application/x-www-form-urlencoded）
void handle_post_form(int client_sock, const char *body) {
    // 解析：device_name=ESP32&firmware=v1.2
    char name[32], version[16];
    if (sscanf(body, "device_name=%31s&firmware=%15s", name, version) == 2) {
        // 业务逻辑：保存配置/触发升级...
        
        // 返回JSON响应
        char json[128];
        snprintf(json, sizeof(json),
                 "{\"code\":200,\"msg\":\"Config saved\",\"device\":\"%s\"}", name);
        send_response(client_sock, 200, "application/json", json);
    } else {
        send_response(client_sock, 400, "application/json", 
                     "{\"code\":400,\"msg\":\"Invalid params\"}");
    }
}
```

> 💡 **嵌入式Server优化建议**：
> ```
> 1. 使用微型框架：mongoose / civetweb（支持SSL/WebSocket/CGI）
> 2. 静态资源压缩：预压缩HTML/CSS/JS，响应时加Content-Encoding: gzip
> 3. 缓存控制：添加Cache-Control头部，减少重复请求
> 4. 安全加固：限制请求频率，过滤危险字符，隐藏服务器版本
> ```

---

## 八、常见问题与调试技巧

### 8.1 高频问题排查表

| 问题现象          | 可能原因                             | 调试方法/解决方案                                            |
| ----------------- | ------------------------------------ | ------------------------------------------------------------ |
| 🔴 连接超时        | 1. DNS解析失败 2. 防火墙拦截         | 1. `ping`测试 2. `telnet host port` 3. 检查`/etc/resolv.conf` |
| 🔴 400 Bad Request | 请求格式错误（换行/空行/头部）       | 用`tcpdump`/`Wireshark`抓包对比标准请求                      |
| 🔴 404 Not Found   | 路径错误/文件权限/根目录配置         | 检查`WEB_ROOT`路径，`ls -l`确认文件存在                      |
| 🔴 响应乱码        | 编码不匹配（服务端UTF-8，客户端GBK） | 统一`Content-Type: ...; charset=utf-8`                       |
| 🔴 JSON解析失败    | 响应包含BOM头/尾部空格/非标准格式    | 打印原始响应，用`cJSON_Parse`前预处理                        |
| 🔴 内存泄漏        | cJSON未Delete / malloc未free         | 用`valgrind`检测，统一资源管理宏                             |

### 8.2 Wireshark过滤表达式（HTTP专项）

```
🔹 基础过滤：
  http                      # 仅显示HTTP协议包
  http.request              # 仅请求包
  http.response             # 仅响应包
  http.request.method == "POST"  # 特定方法

🔹 内容搜索：
  http contains "beijing"   # 搜索报文内容
  http.host == "api.weather.com"  # 特定域名
  http.response.code == 404 # 特定状态码

🔹 性能分析：
  http.time                 # 请求-响应时间
  tcp.analysis.retransmission && http  # HTTP请求重传
```

### 8.3 嵌入式调试技巧

```bash
# 1. 本地测试：用curl模拟客户端
curl -v http://192.168.1.100:8080/api/status
curl -X POST -H "Content-Type: application/json" \
     -d '{"temp":25.6}' http://192.168.1.100:8080/upload

# 2. 开发板抓包（无GUI）：
tcpdump -i eth0 -nn -X port 80 -w /tmp/http.pcap
# 传输到PC分析：
scp root@192.168.1.100:/tmp/http.pcap ./
wireshark http.pcap

# 3. 简化日志：在代码中添加关键节点打印
#define HTTP_DEBUG(fmt, ...) fprintf(stderr, "[HTTP] " fmt "\n", ##__VA_ARGS__)
// 使用：
HTTP_DEBUG("Request: %s %s", method, path);
HTTP_DEBUG("Response: %d bytes, code=%d", body_len, status_code);
```

---

## 九、面试高频考点（嵌入式方向）

```
🔥 HTTP必问题：
1. Q: HTTP和TCP的关系？为什么应用层还需要HTTP？
   A: TCP提供可靠字节流传输，但无应用语义；HTTP定义请求/响应格式、方法、状态码等，
      使客户端/服务器能标准化交互，便于扩展和调试。

2. Q: HTTP无状态，如何实现用户登录会话？
   A: ① Cookie+Session：服务器存会话，客户端传Cookie ② Token（JWT）：客户端存签名令牌
      嵌入式建议：简单设备用Token，避免服务器会话存储压力。

3. Q: GET和POST的本质区别？
   A: ① 语义：GET获取资源（幂等），POST提交数据（非幂等）② 参数位置：GET在URL，POST在正文
      ③ 缓存：GET可缓存，POST默认不缓存 ④ 长度：GET受URL长度限制，POST理论上无限制

4. Q: 嵌入式设备调用HTTP API，如何优化性能和可靠性？
   A: ① Keep-Alive复用连接 ② 设置合理超时 ③ 失败重试+指数退避 ④ 响应压缩（gzip）
      ⑤ 预解析域名 ⑥ 精简请求头部 ⑦ 异步非阻塞（select/epoll）

5. Q: HTTPS在嵌入式设备的实现难点？
   A: ① TLS库体积大（mbedTLS~200KB）② 证书管理复杂 ③ 加解密消耗CPU ④ 随机数生成需真随机源
      解决方案：① 裁剪TLS功能 ② 预置证书 ③ 硬件加速 ④ 用HSM/SE存储密钥

6. Q: 如何解析HTTP响应中的chunked传输编码？
   A: chunked格式：[16进制长度]\r\n[数据]\r\n...[0]\r\n\r\n
      解析步骤：① 读长度行→转十进制 ② 读对应字节数据 ③ 跳过\r\n ④ 循环直到长度=0
      嵌入式建议：优先要求服务端用Content-Length，避免实现chunked解析
```

---

## 📋 协议选型决策树（嵌入式网络通信）

```
🎯 需求：设备需要网络通信
│
├─ 是否需要人类可读/标准接口？
│  ├─ 是 → 进入下一步
│  └─ 否 → 自定义二进制协议（TCP/UDP）+ 极致优化
│
├─ 是否需与Web/云平台对接？
│  ├─ 是 → ✅ HTTP/HTTPS + JSON
│  └─ 否 → 进入下一步
│
├─ 数据是否高频小包（<100B, >1Hz）？
│  ├─ 是 → ✅ MQTT/CoAP（发布订阅+轻量）
│  └─ 否 → 进入下一步
│
├─ 是否需广播/组播？
│  ├─ 是 → ✅ UDP + 自定义协议
│  └─ 否 → ✅ TCP + 自定义协议（可靠+简单）
│
💡 终极建议：
   "标准优先，按需定制" — 能用HTTP/JSON对接云平台时，优先用标准协议降低集成成本；
   资源极端受限时，再考虑自定义轻量协议。
```

---

> ✨ **学习路线建议**：
>
> ```
> 1️⃣ 理解HTTP报文格式 → 2️⃣ 用socket实现GET/POST → 3️⃣ 集成cJSON解析 
> → 4️⃣ 添加超时/重试/错误处理 → 5️⃣ 支持HTTPS（mbedTLS） 
> → 6️⃣ 实现简易HTTP Server → 7️⃣ 压力测试+抓包优化
> ```

> 📌 **核心原则**：  
> **"头部要规范，正文要校验；超时必设置，内存必释放"**

> 🔗 **延伸学习**：
> - 轻量HTTP库：[mongoose](https://github.com/cesanta/mongoose) / [civetweb](https://github.com/civetweb/civetweb)
> - JSON解析：[cJSON](https://github.com/DaveGamble/cJSON)（单文件，易移植）
> - TLS集成：[mbedTLS](https://github.com/Mbed-TLS/mbedtls)（嵌入式友好）
> - 协议对比：MQTT vs CoAP vs HTTP（物联网场景选型）