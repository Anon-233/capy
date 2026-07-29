# Capy 阅读笔记

## 核心设计
1. Capy 的核心设计在于其提出了 Stream 这一概念，并以此为基础构建起一套异步IO处理模型。
2. 从实现上而言，Capy 明确表达了其只面向 coroutine 展开，以此实现简化的效果。
3. Stream 是 Capy 的 IO 抽象核心；IoAwaitable 是 Copy 的协程执行核心。二者在 read_some/write_some 的返回值处结合。

## 案例分析
现在关注到一个具体的例子：include/boost/capy/test/stream.hpp，其中实现了一个简单的 stream 示例。

### 功能简介
1. stream 必须使用 make_stream_pair 创建，其返回一个 pair，元素类型均为 stream。
2. 由此创建的两个 stream 之间可以进行数据传输，即向一个 stream 中写入的数据能够从另一个 stream 中读取。
3. 当没有数据存在时，发起读取的一方会被挂起，直至有数据可读

### 实现解析
1. stream 类只被允许构造，且构造函数被设为私有，其{拷贝/移动}{构造/赋值}函数均被删除
2. 其实现了一组基础的 API 供外部调用
    1. read_some/write_some：用于进行数据的读写操作
3. 其核心数据结构为 stream::state，而 stream::state 的核心数据结构则为两个 stream::half

### std::stop_token 的使用
1. 主要分为 3 个关键部分：
    1. std::stop_source：用于创建后续的 stop_token，并由 stop_source 触发 stop 请求，也即由能够取消任务者首先创建 stop_source，并在后续创建任务时将 stop_token 传入任务中
    2. std::stop_token：用于查询 stop 请求是否被触发，但更重要的是， stop_token 的持有者可以向其注册 stop 触发时的回调函数；为什么需要由任务执行者来注册回调函数呢？因为只有任务执行者才知道如何取消自身
    3. std::stop_callback：用于注册 stop_token 的回调函数，回调函数的执行时机为 stop 请求被触发时，也即在 stop_source 调用 request_stop() 时，stop_token 的回调函数会被同步执行。在行为上表现为取消者调用此前注册的回调函数，发起进一步的取消操作（例如通知 EventLoop 取消任务），而非仅仅局限于任务执行方重复轮询 stop_token 的状态。