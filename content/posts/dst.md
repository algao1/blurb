---
title: "Taming Randomness With Hello World"
date: 2024-07-05
author: "agao"
ShowToc: true
tags: ["simulation testing"]
---

_2024/11/17: Updated to include support for mutexes._

There is a lot of nondeterminism (or randomness) in modern software, from the obvious like random number generation to the less obvious like syscalls, scheduling and network latency. This makes debugging and troubleshooting very troublesome especially in distributed systems where multiple machines and networks are involved.

However, if we could somehow control this randomness, then this would allow us to reproduce issues at will, no matter how rare or difficult. This blog is heavily inspired by the work done by [Reverie](https://github.com/facebookexperimental/reverie), [Hermit](https://github.com/facebookexperimental/hermit), [sled](https://github.com/spacejam/sled) and [Antithesis](https://antithesis.com/). Do check them out.

Another benefit of such a system is that we would able to inject faults and simulate a wide variety of situations. See [chaos engineering](https://netflixtechblog.com/tagged/chaos-engineering) from Netflix for more details.

To control randomness, we will need to simulate all randomness in the system with a pseudo-random generator, and build our systems around it. This is known as **deterministic simulation testing**.

The source code can be found [**here**](https://github.com/algao1/crumbs/tree/master/dst).

![](/blurb/img/dst/dst_simulator.png#center)

## Overview

The basic idea is actually quite simple. We execute the entire program within a single process, so that execution is sequential and deterministic. We also use a pseudo-random **generator** to provide all the randomness in the system. This way, by just specifying a seed, we can reproduce any particular run.

> By running in a single-thread and using a custom scheduler, we can control execution order and make it deterministic. The OS and Goroutine scheduler are nondeterministic. See [here](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html) for more details.

However, because we only have a single thread, we also need to simulate time so that concurrent events occur sequentially. This is effectively what the **timer** does. It is a priority queue ordered by first available time, and all events in the system must be enqueued before being scheduled to execute.

```go
type Event struct {
    T    time.Time
    Task Task
}

type Timer struct {
    CurTime   time.Time // Simulated time.
    Scheduler *TaskScheduler
    Events    EventPriority
}

func (t *Timer) Execute() {
    if len(t.Events) == 0 {
        return
    }
    e := heap.Pop(&t.Events).(*Event)
    t.Scheduler.Schedule(e.Task)
    t.CurTime = e.T
}
```

Each time we execute a task, we advance the `Timer`'s simulated time forward to the current time. In a more complete system, this would allow us to outright ignore things like `time.Sleep` and network latency, and ultimately speed up our simulation. For my initial implementation, I didn't include this, but it shouldn't be too much work to add this (as we'll see later).

Lastly, the **scheduler** is a AIAO (any-in-any-out) queue holding the current list of **runnable** tasks. Using a AIAO queue allows us to execute tasks in a random fashion, so that each run would be different from another.

```go
type TaskScheduler struct {
    Tasks     *list.List
    Generator *Generator
    executed  map[string]int
}

func (s *TaskScheduler) Execute() {
    if s.Tasks.Len() == 0 {
        return
    }
    shifts := s.Generator.Rand() % (s.Tasks.Len())
    cur := s.Tasks.Front()
    for range shifts {
        cur = cur.Next()
    }

    task := s.Tasks.Remove(cur).(Task)
    s.executed[task.Name]++

    if task.Callback() {
        s.Tasks.PushBack(task)
    }
}
```

> Similar to Go's Goroutine scheduler, we consider three states:
>
> - Waiting -- the task is stopped and waiting for something before continuing, this could be `time.Sleep` or a channel `<-someCh`
> - Runnable -- ready and available to be executed
> - Executing -- the task is currently executing

## Our First Hello World

Before delving deeper into the implementation, let's take a look at a very simple example. This program spawns 16 goroutines each printing the string "Hello" followed by "World".

```go
func main() {
    sim := dst.NewSimulator(42)
    for range 16 {
        PrintHelloWorld(sim)
    }
    sim.Run()
}

func PrintHelloWorld(sim *dst.Simulator) {
    sim.Spawn(func(yield func()) {
        fmt.Println("Hello")
        fmt.Println("World")
    })
}
```

We thus get the following results

```
Hello
World
Hello
World
Hello
World
...
```

However, our first results are quite boring since the simulator is not able to preempt a function in the middle of execution. Therefore, "Hello" and "World" will always be printed together. Let's see if we can change that!

## Adding Preemption

There's really no good way of adding preemption without hacking the language runtime itself, or by using more complex methods like what Hermit and Antithesis does. So instead, I've chosen to use **coroutines** to accomplish something similar in effect.

Coroutines allows a task `T` to suspend execution and **yield** control back to our simulator's scheduler and be inserted into the runnable queue again. When it is executed again, the scheduler will **resume** `T`. So, by manually inserting a `yield` in between the print statements, we can achieve a preemption-like effect.

```go
func PrintHelloWorld(sim *dst.Simulator) {
    sim.Spawn(func(yield func()) {
        fmt.Println("Hello")
        yield()
        fmt.Println("World")
    })
}
```

We now get the following

```
Hello
Hello
World
World
Hello
World
...
```

Which is a lot more interesting than before. And if we were to run this multiple times with the same seed, the result would be guaranteed.

```
go run dst/cmd/main.go > dst/out1.txt
go run dst/cmd/main.go > dst/out2.txt
diff dst/out1.txt dst/out2.txt
```

However, to actually implement this is a bit messy. We must make all our tasks (functions) accept a yield function in the form `func(yield func())` so that the task can preempt itself.

The `resume` function wraps around the `yield` function, but only executes it with some probability **P**. This is yet again to simulate randomness in execution order (the task is run immediately after), although we can similarly do this by inserting to the front of the LIRO queue.

Lastly, since the scheduler also needs to know if a task needs to be reinserted after executing, we wrap our `resume` with a `func bool`, and use that as our task.

```go
func (s *Simulator) Spawn(fn func(yield func())) {
    pc, _, _, _ := runtime.Caller(1)
    funcName := runtime.FuncForPC(pc).Name() + fmt.Sprintf("_%d", s.funcCounter)

    resume, _ := coro.New(func(_ bool, yield func(int) bool) int {
        fn(func() {
            if (s.Generator.Rand() % 100) < FUNC_YIELD_PCT {
                slog.Debug("function yielded", slog.String("func", funcName))
                yield(0)
            }
        })
        return 0
    })

    s.Timer.AddEvent(
        func() bool {
            _, ok := resume(false)
            return ok
        },
        funcName,
    )
    s.funcCounter++
}
```

The coroutine package is the one outlined in Russ Cox's [blog](https://research.swtch.com/coro), since at the time of writing, coroutines have not been added to the standard library yet. Do check out the blog for a more comprehensive view of coroutines.

> The `coro` implementation uses goroutines and channels under the hood, but you can ensure that only one process is used by setting `GOMAXPROCS` to 1.

Using the debug stats, we can see that indeed some functions have been preempted and reinserted back into the runnable queue.

```
============== DEBUG STATS ==============
func main.PrintHelloWorld_0 executed 2 times
func main.PrintHelloWorld_1 executed 1 times
func main.PrintHelloWorld_10 executed 2 times
func main.PrintHelloWorld_11 executed 3 times
func main.PrintHelloWorld_12 executed 1 times
func main.PrintHelloWorld_13 executed 1 times
func main.PrintHelloWorld_14 executed 2 times
func main.PrintHelloWorld_15 executed 2 times
func main.PrintHelloWorld_2 executed 2 times
func main.PrintHelloWorld_3 executed 2 times
func main.PrintHelloWorld_4 executed 1 times
func main.PrintHelloWorld_5 executed 1 times
func main.PrintHelloWorld_6 executed 2 times
func main.PrintHelloWorld_7 executed 2 times
func main.PrintHelloWorld_8 executed 2 times
func main.PrintHelloWorld_9 executed 1 times
============== DEBUG STATS ==============
```

## Mutexes

Now that we have the ability to spawn goroutines, we need some way to safely share data between goroutines. Mutexes achieve this by not allowing execution within some critical section if the current thread does not own the lock (or mutex). So the data can only ever be accessed by one process at a given time.

As an example the following code would not generate the correct results without mutexes, since it would be possible for

- goroutine A to increment the counter, then yield control to gorotuine B
- goroutine B increments the counter, prints, returns and yields to A
- goroutine A continnues to print the counter, but with an additional increment

```go
func PrintCounter(sim *dst.Simulator, counter *int) {
    sim.Spawn(func(yield func()) {
        // sim.Lock("counter_lock", yield)
        yield()
        *counter++
        yield()
        fmt.Println(*counter)
        // sim.Unlock("counter_lock")
    })
}
```

```
...
5
6
8
8
9
11
11
12
13
...
```

Ignoring the poor ergonomics (having to name your locks) of using this for a second, the implementation is quite simple. The simulation only needs to keep track of all held locks and **who** they are held by.

```go
type TaskScheduler struct {
    Tasks     *list.List
    Generator *Generator

    // Internals.
    curFunc   string
    heldLocks map[string]string
    executed  map[string]int
}

func (s *TaskScheduler) Lock(lockID string) bool {
    if _, ok := s.heldLocks[lockID]; ok {
        return false
    }
    s.heldLocks[lockID] = s.curFunc
    return true
}

func (s *TaskScheduler) Unlock(lockID string) {
    delete(s.heldLocks, lockID)
}
```

Now, we just need to expose these details to the `Simulator`, and we're done!

> **Q:** In the `Unlock` function, we don't check for the lock's owner before deleting the lock, what are some possible issues with this?

```go
func (s *Simulator) Lock(lockID string, yield func()) {
    for !s.Scheduler.Lock(lockID) {
        yield()
    }
}

func (s *Simulator) Unlock(lockID string) {
    s.Scheduler.Unlock(lockID)
}
```

> **Q:** In the `Lock` function there is a for loop checking if whether the current process (gorotuine) can acquire the lock. Why is this required? Can we do with just an if statement?

Let's try running the previous program and see if it works as expected now.

```
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
```

Great! It prints out the numbers 1 through 16 like we expected, and the debug stats also show a lot of preemption which means the simulator is correctly yielding when the running process can't acquire the lock.

```
============== DEBUG STATS ==============
func main.PrintCounter_0 executed 4 times
func main.PrintCounter_1 executed 2 times
func main.PrintCounter_10 executed 2 times
func main.PrintCounter_11 executed 1 times
func main.PrintCounter_12 executed 1 times
func main.PrintCounter_13 executed 2 times
func main.PrintCounter_14 executed 1 times
func main.PrintCounter_15 executed 2 times
func main.PrintCounter_2 executed 2 times
func main.PrintCounter_3 executed 3 times
func main.PrintCounter_4 executed 2 times
func main.PrintCounter_5 executed 8 times
func main.PrintCounter_6 executed 2 times
func main.PrintCounter_7 executed 2 times
func main.PrintCounter_8 executed 1 times
func main.PrintCounter_9 executed 1 times
============== DEBUG STATS ==============
```

With this done, we should be able to simulate most simple concurrent programs.

## Limitations

However, this approach is not without limitations. For one, doing it on the application level is very intrusive as the user has to manually adjust their program in order to be simulated. This includes

- Manually inserting yields to achieve preemption
- Wrapping concurrent function calls in `sim.Spawn`
- Replacing all sources of randomness with the generator

The current implementation also does not support things like

- `time.Sleep`
- Simulating requests and network latency
- Channels, and other complex synchronization primitives

Implementing the first two should not pose too big of a challenge, but the last would require a more complete environment.

Beyond these issues, there is also a usability issue when debugging and troubleshooting. Because all randomness originates from the simulator and is based on the internal state (of the generator), changing the program slighly may drastically alter the final results even with the same seed.

Take our `Hello World` program for example, if we were to use the generator in between prints (say to determine how long to sleep), we could get drastically different results. This can make it more difficult to verify if our fixes actually resolved the issue or not.

However, we can make up for this by using probability, and by running a ton more simulations. Since every simulation runs in one process, we can run a bunch in parallel with different seeds to give us a better chance of detecting the error (if it still exists).
