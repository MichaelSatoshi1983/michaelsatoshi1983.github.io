---
title: "Go 的 Worker Pool"
date: "2025-06-29 20:27:23"
pubDatetime: 2025-06-29 20:27:23
description: "介绍了 Go 语言中 Worker Pool（工作池）模式的设计与实现 然后构建了一个功能更完善、性能更优的“牛逼 Worker Pool”。"
tags: ["Go"]
---
# 啥玩意是 Worker Pool

Worker 池（Worker Pool）模式是一种**经典的并发设计模式**，通过维护一个固定数量的工作协程（Worker Goroutines）池子来处理任务队列中的任务，从而就可以去达到控制并发数量的目的。和为每个任务创建一个新的 goroutine 相比，Worker Pool 模式能够显著减少资源的消耗还有调度开销，特别是在处理大量轻量级任务时效果对比起来就很明显。

Worker 池模式的核心思想是：**复用有限的工作协程来处理无限的任务流**（一句话就是用固定数量的工作协程，从任务队列里源源不断的取任务来处理，做到有限资源应对无限任务流） ，这使得我们可以有效管理 goroutine 的数量，去构建健壮、可扩展的任务处理系统。在 Go 语言中，由于 goroutine 的创建和销毁成本极低，Worker 池模式的优势可能不如其他语言明显，但在处理大量弟任务时，合理使用去 Worker 池仍然可以带来显著的性能提升。

# 核心组件

一个典型的 Worker 池主要由以下几个核心组件构成：

- **任务队列（Task Queue）**：存储等待处理的任务，一般的话去使用通道（channel）实现。

- **Worker 协程池**：一组固定数量的 goroutine，负责从任务队列中获取任务并执行。

- **任务分发器**：负责将任务添加到任务队列中，并协调 Worker 协程的工作。

Worker 池的工作流程大致如下：

- 初始化创建固定数量的 Worker 协程，并让它们开始监听任务队列。

- 外部任务通过**任务分发器**被添加到任务队列中。

- 空闲的 Worker 协程从任务队列中获取任务然后去执行。

- 任务执行结束后，Worker 协程会继续监听任务队列，然后等待下一个任务。

- 当需要关闭 Worker 池时，所有 Worker 协程会在完成当前任务后退出。

# 实现一个简单的 Worker Pool

## 定义一个任务

首先定义任务类型，任务可以是一个无参数、无返回值的函数：

```go
type Task func()
```
## Worker 池结构体

定义 Worker 池结构体，包含任务队列和 Worker 协程池：

```go
type WorkerPool struct {
    taskQueue chan Task
    workers   []*Worker
    wg        sync.WaitGroup
}

type Worker struct {
    pool *WorkerPool
}
```

## 创建 Worker Pool

创建 Worker 池的函数需要指定 Worker 数量和任务队列容量：

```go
func NewWorkerPool(numWorkers int, taskQueueSize int) *WorkerPool {
    pool := &WorkerPool{
        taskQueue: make(chan Task, taskQueueSize),
    }

    pool.workers = make([]*Worker, numWorkers)
    for i := 0; i < numWorkers; i++ {
        worker := &Worker{pool: pool}
        pool.workers[i] = worker
        pool.wg.Add(1)
        go worker.start()
    }

    return pool
}
```

## Worker 协程启动!!!

每个 Worker 协程启动后开始监听任务队列：

```go
func (w *Worker) start() {
    defer w.pool.wg.Done()
    for task := range w.pool.taskQueue {
        task() // 执行任务
    }
}
```

## 添加任务

向 Worker 池添加任务的方法：

```go
func (p *WorkerPool) Submit(task Task) {
    p.taskQueue <- task
}
```

## 关闭 Worker 池

关闭 Worker 池的方法，确保所有任务完成后再关闭：

```go
func (p *WorkerPool) Shutdown() {
    close(p.taskQueue) // 关闭任务队列，所有Worker协程将退出循环
    p.wg.Wait() // 等待所有Worker协程完成
}
```

# 使用

```go
func main() {
    // 创建包含3个Worker，任务队列容量为10的Worker池
    pool := NewWorkerPool(3, 10)

    // 提交任务
    for i := 0; i < 10; i++ {
        pool.Submit(func() {
            fmt.Println("Task executed")
        })
    }

    // 关闭Worker池
    pool.Shutdown()
}
```

# 分析基础实现

上面的代码简单实现展示了一个 Worker 池的基本结构和工作原理，但存在一些性能和功能上的不足：

- **任务队列阻塞问题**：当任务队列满时，Submit 方法会阻塞，可能影响提交任务的性能。

- **无任务优先级区分**：所有任务按先进先出顺序处理，无法优先处理紧急任务。

- **无 Worker 状态监控**：无法监控 Worker 的运行状态和性能指标。

- **固定 Worker 数量**：无法根据任务负载动态调整 Worker 数量。

- **无任务结果处理**：无法获取任务执行结果或处理任务执行过程中出现的错误。

现在，**让我们来创建一个牛逼一点的 Worker Pool！**

# 牛逼的 Worker Pool

## 任务队列优化

基础实现中的任务队列是一个简单的缓冲通道，当队列满时提交任务会被阻塞。为了提高性能，我们可以实现以下优化：

### 非阻塞任务提交

**修改 Submit 方法，使用非阻塞方式提交任务：**

```go
func (p *WorkerPool) Submit(task Task) bool {
    select {
    case p.taskQueue <- task:
        return true
    default:
        return false // 任务队列已满，提交失败
    }
}
```

这样，当任务队列满时，Submit 方法会立即返回 false，而不是阻塞等待，这在高并发场景下可以避免不必要的阻塞。

## 优先级队列

为了支持任务优先级，可以将任务队列改为优先级队列。使用 Go 标准库中的 container/heap 包实现一个简单的优先级队列：

```go
type PriorityQueue []*TaskWithPriority

type TaskWithPriority struct {
    task     Task
    priority int // 数值越小优先级越高
}

func (pq PriorityQueue) Len() int { return len(pq) }
func (pq PriorityQueue) Less(i, j int) bool { return pq[i].priority < pq[j].priority }
func (pq PriorityQueue) Swap(i, j int) { pq[i], pq[j] = pq[j], pq[i] }

func (pq *PriorityQueue) Push(x interface{}) {
    *pq = append(*pq, x.(*TaskWithPriority))
}

func (pq *PriorityQueue) Pop() interface{} {
    old := *pq
    n := len(old)
    x := old[n-1]
    *pq = old[0 : n-1]
    return x
}
```

**然后在 Worker 池结构体中使用优先级队列：**

```go
type PriorityWorkerPool struct {
    taskQueue PriorityQueue
    workers   []*Worker
    wg        sync.WaitGroup
    mutex     sync.Mutex
    cond      *sync.Cond
}
```

**在 Worker 的 start 方法中从优先级队列获取任务：**

```go
func (w *Worker) start() {
    defer w.pool.wg.Done()
    for {
        task := w.pool.getTask()
        if task == nil {
            break // 任务队列为空且Worker池已关闭
        }
        task.task()
    }
}

func (p *PriorityWorkerPool) getTask() *TaskWithPriority {
    p.mutex.Lock()
    defer p.mutex.Unlock()

    for p.taskQueue.Len() == 0 && !p.closed {
        p.cond.Wait() // 等待任务到来或关闭信号
    }

    if p.taskQueue.Len() == 0 {
        return nil // 任务队列为空且已关闭
    }

    return heap.Pop(&p.taskQueue).(*TaskWithPriority)
}
```

**这样，高优先级的任务会被优先处理，提高了系统的响应速度**

## Worker 动态调整

上面的基础实现中的 Worker 数量是固定的，无法根据任务负载动态调整。为了优化资源利用，可以实现**动态调整 Worker 数量的功能**：

### **动态 Worker 池结构体**

```go
type DynamicWorkerPool struct {
    taskQueue chan Task
    workers   sync.Map // 存储所有Worker，key为Worker ID，value为Worker指针
    workerCount int32 // 当前活动的Worker数量
    maxWorkers int32 // 最大Worker数量
    minWorkers int32 // 最小Worker数量
    wg        sync.WaitGroup
    cond      *sync.Cond
    closed    bool
}
```

### 添加 Worker

```go
func (p *DynamicWorkerPool) addWorker() {
    if atomic.LoadInt32(&p.workerCount) >= p.maxWorkers {
        return
    }

    workerID := atomic.AddInt32(&p.workerCount, 1)
    worker := &Worker{
        pool: p,
        id:   workerID,
    }

    p.workers.Store(workerID, worker)
    p.wg.Add(1)
    go worker.start()
}
```

### 移除 Worker

```go
func (p *DynamicWorkerPool) removeWorker(workerID int32) {
    if atomic.LoadInt32(&p.workerCount) <= p.minWorkers {
        return
    }

    p.workers.Delete(workerID)
    atomic.AddInt32(&p.workerCount, -1)
}
```

### 动态调整策略

```go
func (p *DynamicWorkerPool) adjustWorkers() {
    ticker := time.NewTicker(5 * time.Second) // 每5秒检查一次
    defer ticker.Stop()

    for range ticker.C {
        p.cond.L.Lock()

        taskCount := len(p.taskQueue)
        workerCount := atomic.LoadInt32(&p.workerCount)

        // 根据任务数量调整Worker数量
        if taskCount > 0 && workerCount < p.maxWorkers {
            // 任务积压，增加Worker数量
            newWorkers := int32(math.Min(float64(taskCount), float64(p.maxWorkers-workerCount)))
            for i := 0; i < int(newWorkers); i++ {
                p.addWorker()
            }
        } else if taskCount == 0 && workerCount > p.minWorkers {
            // 任务队列为空，减少Worker数量
            workersToRemove := int32(math.Max(float64(workerCount-p.minWorkers), 0))
            p.workers.Range(func(key, value interface{}) bool {
                if workersToRemove > 0 {
                    workerID := key.(int32)
                    p.workers.Delete(workerID)
                    workersToRemove--
                }
                return workersToRemove > 0
            })
            atomic.AddInt32(&p.workerCount, -workersToRemove)
        }

        p.cond.L.Unlock()
    }
}
```

## 任务结果处理与错误处理

### 带结果和错误的任务定义

> 我比较懒，所以这一段的错误处理就简单的带过~~~

```go
type Task func() (interface{}, error)

type Result struct {
    taskID int
    result interface{}
    err    error
}
```

### 带结果处理的 Worker 池

```go
type ResultWorkerPool struct {
    taskQueue chan *TaskWithID
    resultChan chan *Result
    workers   []*Worker
    wg        sync.WaitGroup
}

type TaskWithID struct {
    id int
    task Task
}
```

### Worker 执行任务并返回结果

```go
func (w *Worker) start() {
    defer w.pool.wg.Done()
    for taskWithID := range w.pool.taskQueue {
        result, err := taskWithID.task()
        w.pool.resultChan <- &Result{
            taskID: taskWithID.id,
            result: result,
            err:    err,
        }
    }
}
```

### 获取任务结果

```go
func (p *ResultWorkerPool) GetResult() <-chan *Result {
    return p.resultChan
}
```

# 总结

通过上面的一些优化，我们的Worker Pool已经变得相对来说比较优雅了。不过篇幅有限，这里就不继续去深入了，等我后续慢慢补上~

下面我们来探讨一下怎么正确的用

# 最佳实践

## 限制数量

设置合适的 Worker 数量对于 Worker 池的性能至关重要。比如根据 CPU 核心数设置：对于 CPU 密集型任务，可以根据 CPU 核心数设置 Worker 数量，这样可以充分利用 CPU 资源，避免 CPU 核心空闲。

**还可以根据根据任务类型动态调整**：对于 I/O 密集型任务，可以适当增加 Worker 数量，因为在等待 I/O 操作时，Worker 协程可以处理其他任务。

当然还可以根据根据系统负载动态调整，这里就不细说了。

## 任务队列大小设置

**任务队列大小会影响 Worker 池的性能和内存使用：**对于处理速度较快的任务，可以设置较小的任务队列。

# 真正的总结

Worker 池模式作为一种经典的并发设计模式，具有以下优势：

1. **资源控制**：可以有效控制并发数量，避免系统资源耗尽。

2. **性能优化**：减少 goroutine 创建和销毁的开销，提高任务处理效率。

3. **负载均衡**：所有任务可以均匀分配给各个 Worker 协程，避免有的 Worker 过度繁忙，有的则空闲。

4. **可管理性**：提供了统一管理和监控任务执行的接口（等我后面加上），便于系统维护和故障排查。

但是，Worker 池模式也存在一些局限：

- **复杂性增加**：实现一个功能完善的 Worker 池需要较多的代码，增加了系统的复杂性。

- **资源竞争**：在高并发场景下，任务队列和共享资源一定会成为成为性能的瓶颈。

- **灵活性降低**：固定的 Worker 数量可能无法适应任务负载的剧烈变化。

- **调试难度增加**：由于任务在多个 goroutine 中并行执行，调试和排查问题变得更加困难。（也就是不可观测。。。）

不过，通过合理的使用worker pool模式，我们可以构建出高效、稳定、可扩展的并发系统，从而充分的去发挥我们 Go 语言的并发优势。