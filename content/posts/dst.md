---
title: "Taming Randomness With Hello World"
date: 2024-07-05
author: "agao"
ShowToc: true
---

There is a lot of nondeterminism (or randomness) in modern software, from the obvious like random number generation to the less obvious like syscalls, scheduling and network latency. This makes debugging and troubleshooting very troublesome especially in distributed systems where multiple machines and networks are involved.

However, if we could control this randomness, then this would allow us to reproduce issues at will, no matter how rare or difficult. This blog is heavily inspired by the work done by [Reverie](https://github.com/facebookexperimental/reverie), [Hermit](https://github.com/facebookexperimental/hermit), [sled](https://github.com/spacejam/sled) and [Antithesis](https://antithesis.com/). Do check them out.

Another benefit of such a system is that we are able to inject faults and simulate a wide variety of situations. See [chaos engineering](https://netflixtechblog.com/tagged/chaos-engineering) from Netflix for more details.

To control randomness, we will instead simulate all randomness in the system with a pseudo-random generator, and build a harness around it. This is known as **deterministic simulation testing**.

The source code can be found [**here**](https://github.com/algao1/crumbs/tree/master/dst).

![](/blurb/img/dst/dst_simulator.png)

## Overview

The basic idea is actually quite simple. We execute the entire program within a single process, so execution is sequential and deterministic. We also use a pseudo-random **generator** to provide all other randomness in the system. This way, by just specifying a seed, we can reproduce any single run.

> By running in a single-thread and using a custom scheduler, we can control execution order and make it deterministic. The OS and Goroutine scheduler are nondeterministic. See [here](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html).

However, because we only have a single thread, we also need to simulate time so that concurrent events occur sequentially. This is effectively what the **timer** does. It is a priority queue ordered by first available time, and all events in the system must be enqueued before being scheduled to execute.

```go
type Event struct {
    T    time.Time
    Task Task
}
```

In a more complete system, this would allow us to outright ignore things like `time.Sleep` and network latency, and ultimately speed up our simulation. For my first implementation, I didn't include this, but it shouldn't be too much work to add this (as we'll see later).

Lastly, the **scheduler** is a LIRO (last-in-random-out) queue holding the current list of **runnable** tasks. Using a LIRO queue allows us to execute tasks in a random fashion, so that each run would be different from another.

> Similar to Go's Goroutine scheduler, we consider three states:
>
> - Waiting -- the task is stopped and waiting for something before continuing, this could be `time.Sleep` or a channel `<-someCh`
> - Runnable -- ready and available to be executed
> - Executing -- the task is currently executing

## Our First Hello World

Before delving deeper into the implementation, let's take a look at a very simple example. This program spawns 16 processes each printing the string "Hello\nWorld\n".

```go
func main() {
    slog.SetLogLoggerLevel(slog.LevelDebug)
    sim := dst.NewSimulator(42)
    // Equivalent to spawning 16 threads/goroutines
    // and printing "Hello World" on each.
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

We get the following results

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

There's really no good way of adding preemption without hacking the language itself, or by using more complex methods like what Hermit and Antithesis does. So instead, I've chosen to use **coroutines** to accomplish something similar in effect.

Coroutines allows a task `T` to suspend execution and _yield_ back to our simulator's scheduler and be inserted into the runnable queue again. When it is executed again, the scheduler will _resume_ `T`.

So, by manually inserting a `yield` in between the print statements, we can achieve a preemption-like effect.

```go
func PrintHelloWorld(sim *dst.Simulator) {
    sim.Spawn(func(yield func()) {
        fmt.Println("Hello")
        yield()
        fmt.Println("World")
    })
}
```

We now get

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

However, to actually implement this is a bit messy. We must make all our tasks (`fn`) accept a yield function in the form `func(yield func())` so that the task can preempt itself.

The `resume` function wraps around the `yield` function, but only executes it with some probability `P` (`P=0.3` in this case). This is yet again to simulate more randomness in execution order (the task is run immediately after), although we can similarly do this by inserting to the front of the LIRO queue.

Lastly, since the scheduler needs to know if a task is to be reinserted after executing, we wrap our `resume` with a `func bool`, and use that as our task.

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

## Limitations

However, this approach is not without limitations. For one, doing it on the application level means that the user has to manually adjust their program in order to be simulated. This includes

- Manually inserting yields to achieve preemption
- Wrapping concurrent function calls in `sim.Spawn`
- Replacing all sources of randomness with the generator

The current implementation also does not support things like

- `time.Sleep`
- Simulating requests and network latency
- Channels, mutexes, and most synchronization primitives

Implementing the first two should not pose too big of a challenge, but the last would require a more complete environment.

Beyond these issues, there is also a usability issue when debugging and troubleshooting. Because all randomness originates from the simulator and is based on the internal state (of the generator), if we were to change the program slightly, we may get completely different results even with the same seed.

Take our `Hello World` program for example, if we were to use the generator in between prints, we could get drastically different results. This makes it more difficult to verify if our fixes actually resolved the issue or not.

However, we can make up for this by using probability, and running a ton more simulations. Since every simulation runs in one process, we can run a bunch in parallel with different seeds to give us a better chance of detecting the error (if it still exists).
