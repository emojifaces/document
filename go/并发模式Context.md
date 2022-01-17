# 并发模式 Context

在 Go 服务中，每个传入的请求在单独的 **goroutine** 中处理。请求回调函数通常启动额外的 **goroutine** 以访问后端，如数据库和 RPC 服务。处理同一请求的一系列 **goroutine** 通常需要访问请求相关的值，例如端用户的标识、授权令牌和请求截止时间。当请求被取消或超时，处理该请求的所有 **goroutine** 都应该快速退出，以便系统可以回收它们正在使用的资源。

在 Google，我们开发了一个上下文包，可以轻松地跨越 API 边界，将请求作用域内的值、取消信号和截止时间传递给所有处理请求的 **goroutine**。

## Context
context 包的核心是 **Context** 类型
```
type Context interface {
    Done() <-chan struct{}

    Err() error

    Deadline() (deadline time.Time, ok bool)

    Value(key interface{}) interface{}
}
```
Done 方法返回一个 channel，充当传递给 Context 下运行的函数的取消信号：当 channel 关闭时，函数应该放弃它们的工作并返回。Err 方法返回一个错误，表明取消 context 的原因。

Context 没有 Cancel 方法，原因与 Done channel 是只读的一样：接收取消信号的函数通常不是发送信号的函数。特别是当父操作为子操作启动 goroutine 时，子操作不应该有能力取消父操作。相反，WithCancel 函数（如下所述）提供了一种取消新 Context 值的方法。

多个 goroutine 同时使用同一 Context 是安全的。代码可以将单个 Context 传递给任意数量的 goroutine，并取消该 Context 以向所有 goroutine 发送信号。

Deadline 方法允许函数决定是否应该开始工作；如果剩下的时间太少，则可能不值得。代码还可以使用截止时间来设置 I/O 操作超时。

Value 允许 Context 携带请求作用域的数据。为使多个 goroutine 同时使用，这些数据必须是安全的。

## Derived contexts

context 包提供了从现有 Context 值派生新 Context 值的函数。这些值形成一个树：当 Context 被取消时，从它派生的所有 Context 也被取消。

Background 是所有 Context 树的根；它永远不会被取消

WithCancel 和 WithTimeout 返回派生 Context 值，可以比父 Context 更早取消。当请求回调函数返回时，通常会取消与传入请求关联的 Context。WithCancel 还可用于使用多个副本时取消冗余的请求。WithTimeout 用于设置对后端服务器请求的截止时间

WithValue 提供了一种将请求作用域的值与 Context 关联的方法