---
title: C++三方库
top_img: transparent
date: 2022-09-19 21:57:30
updated: 2022-09-19 21:57:30
tags:
  - Linux
  - C++
  - Redis
categories: C++
keywords:
description: 一些好用的c++库
---

## fmt

格式化字符串，好像已经纳入C++20标准库，使用很方便！而且说是比`printf`还快~

```bash
git clone --depth=1 https://github.com/fmtlib/fmt.git

cd fmt/
mkdir build
cd build
cmake ..
make -j

sudo make install
# g++ test.cpp -o debug -lfmt
```

> [github 仓库，README中文档也较详细、使用简单](https://github.com/fmtlib/fmt)
>
> [官方文档、有代码示例](https://hackingcpp.com/cpp/libs/fmt.html)

## fmtlog

基于`fmt`的日志库，可以很方便的以`fmt`的方式格式化日志输出。

```bash
git clone https://github.com/MengRao/fmtlog.git

cd fmtlog
git submodule init
git submodule update
./build.sh

# g++ test.cpp -o debug -lfmt -lfmtlog-static # 好像只能使用静态库
```

> [官方仓库，也只有这里的README作为官方文档](https://github.com/MengRao/fmtlog)

## spdlog

同样支持`fmt`格式输出的日志库，相对更成熟一点。

https://zhuanlan.zhihu.com/p/427038912

https://www.modb.pro/db/251872

## hiredis

`C`语言版本的`Redis Client`！

```bash
git clone --depth=1 https://github.com/redis/hiredis.git

cd hiredis
make

sudo make install
# g++ test.cpp -o debug -lhiredis
```

> [官方仓库，也只有这里的README作为官方文档](https://github.com/redis/hiredis)
>
> [hiredis.h 头文件，可以看 redis 的API](https://github.com/redis/hiredis/blob/79ae5ffc693b57688b4c76141fd2c94868ebdbff/hiredis.h#L305)
>
> [别人翻译的官方文档，供参考](https://tangming.github.io/2019/10/14/redis-hiredis-introduction/)

Hiredis提供了同步、异步以及回复解析三种API。

### 同步API

要使用同步 API，只需要引入几个函数调用：

```c
/* 两个重要结构体 */
redisContext *ctx;  // 连接上下文，建立连接时创建
redisReply *reply;  // redisCommand返回值，注意释放内存

/* 和redis服务器建立TCP连接 */
redisContext *redisConnect(const char *ip, int port);
redisContext *redisConnectWithTimeout(const char *ip, int port, const struct timeval tv);

/* 发送指令到redis服务器，并取得结果 */
void *redisCommand(redisContext *c, const char *format, ...);
void *redisCommandArgv(redisContext *c, int argc, const char **argv, const size_t *argvlen);

/* 释放内存 */
void freeReplyObject(void *reply);
void redisFree(redisContext *c);    // 断开连接，关闭套接字
```

#### redisConnect

`Hiredis`通过`redisConnect`创建一个`redisContext`来实现与`Redis`进行连接，`context`中包含了连接的信息。`redisContext`中包含有一个整形的`err`变量和一个字符类型的`errstr`变量，当创建连接失败，`err`为非零值，`errstr`为错误的表述。**当使用`redisConnect`创建连接后，应该检查`err`参数以判断连接是否成功**。

```c
redisContext *c = redisConnect("127.0.0.1", 6379);
if (c == NULL || c->err) {
    if (c) {
        printf("Error: %s\n", c->errstr);
        // handle error
    } else {
        printf("Can't allocate redis context\n");
    }
}

/* 使用 timeout */
struct timeval timeout = {2, 0};    // {s, us};
redisContext *c = redisConnectWithTimeout("127.0.0.1", 6379, timeout);
```

#### redisCommand

给数据库发送指令，指令与通过`redis-cli`使用时一致~

```c
reply = redisCommand(context, "SET foo bar");
reply = redisCommand(context, "SET foo %s", value)
reply = redisCommand(context, "SET foo:%s %s", key, value);
reply = redisCommand(context, "GET key");

/* 注意每次调用 redisCommand 都需要 free(reply) */
freeReplyObject(reply);
```

#### redisCommandArgv

批量执行命令！

```c
// argv: 命令字符串数组， {"SET FOO 1", "GET FOO", "SET FOO 2", "GET FOO"};
// argc: 命令argv的数量
// argvlen: agrv中每个字符串的长度数组 {9, 7, 9, 7}。argvlen为NULL时，则调用strlen计算长度
// 返回值：reply 指针，和redisCommand一致。注意此时reply中包含多个返回值，位于reply->element数组中
void *redisCommandArgv(redisContext *c, int argc, const char **argv, const size_t *argvlen);
```

## redis-plus-plus

基于`hiredis`实现的`c++`版本的`redis`客户端~

```bash
git clone --depth=1 https://github.com/sewenew/redis-plus-plus.git

cd redis-plus-plus
mkdir build
cd build
cmake -DREDIS_PLUS_PLUS_CXX_STANDARD=17 ..
make

sudo make install
# g++ test.cpp -o debug -lhiredis -lredis++
```

> [github 仓库，其README中有较详细的文档](https://github.com/sewenew/redis-plus-plus)

## cpp-httplib

可创建`HTTP Server`，也可创建`HTTP Client`用于发送`get/post`等请求。

相关方法好像都在`httplib.h`这个头文件中。

```bash
git clone --depth=1 https://github.com/yhirose/cpp-httplib.git

cd cpp-httplib
mkdir -p build
cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
sudo cmake --build . --target install

# 好像只需要将httplib.h拷贝到include目录下就行
s@ys:build$ sudo cmake --build . --target install
Install the project...
-- Install configuration: "Release"
-- Installing: /usr/local/include/httplib.h
-- Installing: /usr/local/lib/cmake/httplib/httplibConfig.cmake
-- Installing: /usr/local/lib/cmake/httplib/httplibConfigVersion.cmake
-- Installing: /usr/local/lib/cmake/httplib/FindBrotli.cmake
-- Installing: /usr/local/lib/cmake/httplib/httplibTargets.cmake
```

> [httplib.h 直接看该文件找API](https://github.com/yhirose/cpp-httplib/blob/master/httplib.h)
>
> [一点解读](https://segmentfault.com/a/1190000022419921)

常用类和一些结构：

```c++
class Server;       // 服务端类
class ThreadPool;   // 线程池类
class Client;       // 客户端类
struct Request;     // 请求数据类
class Result;       // 请求返回类，包含 Response
struct Response;    // 响应数据类

class Result {
    std::unique_ptr<Response> res_;
    Error err_;
    Headers request_headers_;
}

struct Response {
    std::string version;
    int status = -1;        // 状态位，200/403...
    std::string reason;
    Headers headers;
    std::string body;       // 响应 body
    std::string location;   // Redirect location
}
```

### 客户端

```c++
#include <httplib.h>

/* 创建Client对象 */
explicit Client(const std::string &scheme_host_port);
httplib::Client cli("http://192.168.1.147:18181");

explicit Client(const std::string &host, int port);
httplib::Client cli("http://192.168.1.147", 18181);
```

#### Get请求

```c
/* get 请求 */
httplib::Headers headers = {{ "Authorization", "Basic b25vczpyb2Nrcw==" }};

Result Get(const std::string &path);
Result Get(const std::string &path, const Headers &headers);
Result res = cli.Get("/onos/v1/flows/of:0000000000001111", headers);

cli.set_default_headers(headers);   // 设置默认Header
Result res = cli.Get("/onos/v1/flows/of:0000000000001111");
```

#### 返回值

```c
Result res = cli.Get("/onos/v1/flows/of:0000000000001111");

/* Result重载了 bool() 和 ->，可直接使用->访问Response的值 */
if (res) {
    cout << res->status << endl;
    cout << res->get_header_value("Content-Type") << endl;
    cout << res->body << endl;
} else {
    cout << res.error() << endl;
    auto err = res.error();
    std::cout << "HTTP error: " << httplib::to_string(err) << std::endl;
}
```

#### Post请求

```c
Result Post(const std::string &path);
Result Post(const std::string &path, const std::string &body,
            const std::string &content_type);
Result Post(const std::string &path, const Headers &headers,
            const std::string &body, const std::string &content_type);
```

## json

解析json字符串，返回json对象。

```bash
git clone --depth=1 https://github.com/nlohmann/json.git

cd json/
mkdir build
cd build
cmake ..
make -j
sudo make install
```

> [文档 API](https://json.nlohmann.me/api/basic_json/)
