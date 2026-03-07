---
layout: post
title: Go 并发编程实战：从 goroutine 泄漏到优雅退出
categories: Go
description: 实习和自学过程中踩过的 Go 并发坑，整理出一些实用的并发模式。
keywords: Golang, goroutine, concurrency, channel, context, errgroup
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## 一、goroutine 泄漏：你以为它退出了，其实没有

最经典的场景：往一个无缓冲 channel 发数据，但没人收。

```go
func process(ctx context.Context) {
    ch := make(chan result)
    go func() {
        r := doSomethingExpensive()
        ch <- r // 如果外面没人读 ch，这个 goroutine 永远阻塞在这里
    }()

    select {
    case <-ctx.Done():
        return // 超时了，直接返回，但上面的 goroutine 泄漏了
    case r := <-ch:
        handleResult(r)
    }
}
```

**修复方案：用带缓冲的 channel。**

```go
ch := make(chan result, 1) // 缓冲区为 1
```

这样即使没人读，goroutine 也能写入后退出。这个问题表现为内存缓慢增长，pprof 一看几千个 goroutine 卡在 channel send。

还有一种更隐蔽的泄漏——往已关闭的 HTTP 连接写响应：

```go
go func() {
    resp, err := http.Get(url)
    if err != nil {
        return
    }
    defer resp.Body.Close()
    // 如果忘了 defer resp.Body.Close()，底层连接不会归还连接池
    // goroutine 虽然退出了，但 TCP 连接泄漏
    body, _ := io.ReadAll(resp.Body)
    ch <- body
}()
```

**经验：跑测试前一定用 `goleak` 检测。**

```go
func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m)
}
```

## 二、context 是并发控制的骨架

刚开始写代码的时候觉得 context 就是传个超时，后来才意识到它是整个请求链路的生命线。

### 超时传播

```go
func HandleRequest(w http.ResponseWriter, r *http.Request) {
    // 整个请求最多 5 秒
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()

    userInfo, err := getUserInfo(ctx, userID)
    if err != nil {
        // 这里 err 可能是 context.DeadlineExceeded
        http.Error(w, "timeout", http.StatusGatewayTimeout)
        return
    }
    orders, err := getOrders(ctx, userID)
    // ...
}
```

重要的是 `getUserInfo` 和 `getOrders` 内部也要尊重 ctx：

```go
func getUserInfo(ctx context.Context, id string) (*User, error) {
    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
    resp, err := client.Do(req)
    // ctx 取消时，底层连接会被中断，不会傻等
    // ...
}
```

### 常见错误：存了一个已取消的 context

```go
type Server struct {
    ctx context.Context // 千万别这么干
}
```

context 应该通过函数参数传递，而不是存在 struct 里。存在 struct 里意味着它的生命周期和 struct 绑定，而不是和请求绑定。

## 三、errgroup：并发请求的标准姿势

之前写并发请求是手动管 WaitGroup + error channel，代码又长又容易出错。后来发现 `errgroup` 基本是标配：

```go
func fetchUserProfile(ctx context.Context, uid string) (*Profile, error) {
    g, ctx := errgroup.WithContext(ctx)

    var user *User
    var orders []*Order
    var avatar string

    g.Go(func() error {
        var err error
        user, err = getUser(ctx, uid)
        return err
    })

    g.Go(func() error {
        var err error
        orders, err = getOrders(ctx, uid)
        return err
    })

    g.Go(func() error {
        var err error
        avatar, err = getAvatar(ctx, uid)
        return err
    })

    if err := g.Wait(); err != nil {
        return nil, err
    }

    return &Profile{User: user, Orders: orders, Avatar: avatar}, nil
}
```

`errgroup.WithContext` 的好处：任何一个 goroutine 返回 error，ctx 会被取消，其它 goroutine 可以快速退出。

### 限制并发数

实际项目中经常需要控制并发度，比如批量调用下游接口不能一次打太多：

```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(10) // 最多 10 个并发

for _, item := range items {
    item := item // Go 1.22 之前需要这行
    g.Go(func() error {
        return processItem(ctx, item)
    })
}

if err := g.Wait(); err != nil {
    return err
}
```

## 四、Worker Pool 模式

对于持续消费队列消息的场景，我一般用这个模式：

```go
func StartWorkerPool(ctx context.Context, queue <-chan Task, concurrency int) error {
    g, ctx := errgroup.WithContext(ctx)

    for i := 0; i < concurrency; i++ {
        g.Go(func() error {
            for {
                select {
                case <-ctx.Done():
                    return ctx.Err()
                case task, ok := <-queue:
                    if !ok {
                        return nil // channel 关闭，正常退出
                    }
                    if err := processTask(ctx, task); err != nil {
                        log.Printf("process task failed: %v", err)
                        // 单个任务失败不退出整个 pool
                        // 如果要 fail-fast，直接 return err
                    }
                }
            }
        })
    }

    return g.Wait()
}
```

关键点：
- 用 `ctx.Done()` 控制退出，不用额外的 quit channel
- channel 关闭时 worker 自动退出
- 单个任务失败是否要拖垮整个 pool，根据业务决定

## 五、sync.Map 不是万能的

初学的时候看到并发 map 读写会 panic，第一反应就是用 `sync.Map`。但实际上 `sync.Map` 只适合两种场景：
1. key 只写入一次，后续大量读
2. 多个 goroutine 读写不同的 key

对于频繁读写相同 key 的场景，`sync.RWMutex` + 普通 map 性能更好：

```go
type SafeCache struct {
    mu    sync.RWMutex
    items map[string]*CacheItem
}

func (c *SafeCache) Get(key string) (*CacheItem, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    item, ok := c.items[key]
    return item, ok
}

func (c *SafeCache) Set(key string, item *CacheItem) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[key] = item
}
```

做过一次 benchmark，在高并发读写同一批 key 的场景下，`sync.RWMutex` 方案比 `sync.Map` 快了约 40%。

## 六、优雅退出

实际的服务不能直接 kill，需要处理完正在进行的请求再退出：

```go
func main() {
    srv := &http.Server{Addr: ":8080", Handler: router}

    go func() {
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatalf("listen: %v", err)
        }
    }()

    // 等待中断信号
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    log.Println("shutting down...")

    // 给 30 秒处理完正在进行的请求
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Fatalf("server shutdown: %v", err)
    }

    log.Println("server exited")
}
```

如果还有后台 worker，需要配合退出：

```go
<-quit

// 1. 停止接收新请求
srv.Shutdown(ctx)

// 2. 停止消费新消息
close(taskQueue)

// 3. 等 worker pool 处理完
workerPool.Wait()

// 4. 关闭数据库连接
db.Close()
```

**顺序很重要：先停入口，再等任务完成，最后关资源。反过来会导致正在处理的请求报错。**

## 总结

| 场景 | 推荐方案 |
|------|---------|
| 并发请求聚合 | `errgroup` |
| 持续消费任务 | Worker Pool + context |
| 并发安全 map | 高频读写同 key 用 `sync.RWMutex`，否则 `sync.Map` |
| 超时控制 | `context.WithTimeout` 链式传递 |
| 泄漏检测 | `goleak` + pprof |
| 优雅退出 | `signal.Notify` + `Shutdown` |

这些模式基本覆盖了我日常 80% 的并发场景。不追求花哨，够用、稳定、好排查问题最重要。
