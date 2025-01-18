---
layout: post
title:  "Throttling dispatcher in Go"
date:   2019-09-30 20:27:42 +0200
categories: go concurrency throttling dispatcher
---
One evening I was thinking why don't I implement throttling dispatcher in Go. I even had to find ["The Go Programming Language"][gopl] book on the shelf, it was waiting for this for more than a year.

While Go has a lot of controversy in it (like, [no generics][go-generics], at least for now, and weird/missing dependency management), I must admit it is very simple and very powerful language with "batteries included": advanced concurrency primitives like goroutines and channels are built into the language, as well as a lot of useful utilities.

Feels like it was created to quickly produce unsophisticated code - calling/service API (e.g. gRPC), intergations, utilities, etc. But even quite meaningful throttling dispatcher that I'll cover here takes just under 70 lines of code.

The dispatcher is responsible for (eventually) running the submitted task. Throttling in this case is just limiting parallelism - number of simultaneously running tasks.

```go
type Scheduler interface {
    submit(f func())
    stop()
}
```

Non-throttling dispatcher is essentially just a (bounded) queue of tasks waiting to be executed:

```go
type NonThrottlingScheduler struct {
    tasks chan func()
}

func NewScheduler() *NonThrottlingScheduler {
    tasks := make(chan func(), 1024)
    go func() {
        for {
            task, ok := <-tasks
            if !ok {
                break
            }
            go task()
        }
    }()

    return &NonThrottlingScheduler{
        tasks: tasks,
    }
}

func (s *NonThrottlingScheduler) submit(f func()) {
    s.tasks <- f
}

func (s *NonThrottlingScheduler) stop() {
    close(s.tasks)
}
```

Nice thing to have in the scheduler is an opportunity to wait until all tasks have completed their execution. Without it, following program introduces race condition and can finish before any of the tasks get actually completed:

```go
func main() {
    scheduler := NewScheduler()
    for i := 0; i < 10; i++ {
        id := i // capture the index
        scheduler.submit(func () {
            fmt.Printf("%v working...\n", id)
        })
        fmt.Printf("submitted %v\n", id)
    }

    scheduler.stop()
    fmt.Printf("completed\n")
}
```

```bash
$ go run main.go
submitted 0
submitted 1
submitted 2
submitted 3
submitted 4
submitted 5
submitted 6
submitted 7
submitted 8
submitted 9
completed
```

To avoid race condition, I'm introducing a new channel, that will only get the object put there when all tasks have finished. Then waiting on this channel will allow waiting for all tasks to complete. Closing the tasks channel before waiting for completing might be a good idea!

```go
type Scheduler interface {
    submit(f func())
    stop()
    await()
}

type NonThrottlingScheduler struct {
    tasks chan func()
    done chan struct{}
}

func NewScheduler() *NonThrottlingScheduler {
    var wg sync.WaitGroup
    done := make(chan struct{})

    tasks := make(chan func(), 1024)
    go func() {
        for {
            task, ok := <-tasks
            if !ok {
                break
            }
            wg.Add(1)   // increment the counter when task is received
            go func() {
                defer wg.Done() // decrement when taks is completed
                task()
            }()
        }

        wg.Wait()
        done <- struct{}{}
    }()

    return &NonThrottlingScheduler{
        tasks: tasks,
        done: done,
    }
}

func (s *NonThrottlingScheduler) await() {
    <-s.done
}
```

Now program finishes only when all tasks complete:

```go
$ go run main.go
submitted 0
submitted 1
submitted 2
submitted 3
submitted 4
submitted 5
submitted 6
submitted 7
submitted 8
0 working...
submitted 9
2 working...
8 working...
1 working...
5 working...
6 working...
7 working...
4 working...
3 working...
9 working...
completed
```

As for throttling, there is a separate channel for tokens, initiated when schedule is created. Then a task is only gets running when the token is available. Respectively, when a task is completed, the token must be returned back to the channel.

```go
type ThrottlingScheduler struct {
    tokens chan struct{}
    tasks chan func()
    done chan struct{}
}

func NewThrottlingScheduler(maxParallelism int, maxQueueLength int) *ThrottlingScheduler {
    done := make(chan struct{})

    tokens := make(chan struct{}, maxParallelism)
    for i := 0; i < maxParallelism; i++ {
        tokens <- struct{}{}
    }

    var wg sync.WaitGroup
    tasks := make(chan func(), maxQueueLength)
    go func() {
        for {
            task, ok := <-tasks
            if !ok {
                break
            }
            token := <-tokens
            wg.Add(1)
            go func() {
                defer wg.Done()
                defer func() {
                    tokens <- token
                }()
                task()
            }()
        }
        wg.Wait()
        close(tokens)
        done <- struct{}{}
    }()

    return &ThrottlingScheduler {
        tokens: tokens,
        tasks: tasks,
        done: done,
    }
}

func run(s Scheduler, numTasks int) {
    for i := 0; i < numTasks; i++ {
        idx := i
        s.submit(func () {
            fmt.Printf("%v working...\n", idx)
            time.Sleep(1 * time.Second)
        })
        fmt.Printf("submitted %v\n", idx)
        time.Sleep(100 * time.Millisecond)
    }

    s.stop()
    s.await()
    fmt.Printf("completed\n")
}
```

Full throttling scheduler in under 100 lines of code, readable and understandable. Go is not that bad as it seems. Full code is on [GitHub][code].

[gopl]: https://www.gopl.io/
[go-generics]: https://blog.golang.org/why-generics
[code]: https://github.com/sergey-melnychuk/throttling-dispatcher-in-go
