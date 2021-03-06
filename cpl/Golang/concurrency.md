---
layout: default
---
# Concurrency
Concurrency is the ability for a program to do multiple things
at the 'same' time.

When parts of code are running concurrently, you are often unable to determine
when things will happen and in what order.

## Goroutines
A goroutine is a lightweight thread of execution managed by the Go runtime.
Different from OS threads, they are independent, concurrent threads
of control which share the same address space.

When [main()] returns, the program exits. It does not wait for other (non-main)
goroutines to complete. This is fundamentally different than the model used in
Node.js.

### The `go` Keyword
A `go` statement starts the execution of a function call as a goroutine.
Unlike a regular call, program execution does not wait for the invoked
function to complete, as mentioned above.

```go
package main
import(
  "fmt"
  "time"
)

func say(s string) {
  for i:=0; i < 5; i++ {
    fmt.Print(s)
    time.Sleep(500 * time.Millisecond)
  }
}

func main() {
  go say("Hello ")
  time.Sleep(0.5 * time.Second)
  for i := 0; i < 5; i++ {
    fmt.Println("world!")
    time.Sleep(1 * time.Second)
  }
}
```
##### Output
```
Hello world!
Hello world!
Hello world!
Hello world!
Hello world!
```


## Channels
Channels are a primitive needed in order to communicate between goroutines.
You can send/recieve over a channel, in a first-in-first-out queue.
Channels can be buffered or unbuffered. An unbuffered channel-communication
succeeds **only** when a sender *and* receiver are both ready. In contrast,
a buffered channel succeeds without blocking if the buffer is not full when
sending, or not empty in the case of receiving.

### Quick Facts:
- A channel's zero-value is `nil`
- Channels must be initialized with `make()`

### Example(unbuffered):
```go
func print(strings chan string) {
  for {
    x := <- strings
    fmt.Println(x)
    // what follows is the idiomatic method:
    // fmt.Println(<- strings)
  }
}

func main() {
  c := make(chan string)
  go print(c)
  c <- "apple"
  c <- "banana"
  c <- "carrot"
}
```

### Example(buffered)
```go
c := make(chan string, IO)
c <- "apple"
c <- "banana"
c <- "carrot"

fmt.Println(<- c)
fmt.Println(<- c)
fmt.Println(<- c)
```

### Example with Concurrent Data Processing:
```go
package main
import (
  "bufio"
  "fmt"
  "io"
  "os"
)

func countWords(name string, success chan bool) {
  file, err := os.Open(name)
  if err != nil {
    fmt.Println(name, err)
    success <- false
    return
  }
  defer file.Close()
  scanner := bufio.NewScanner(file)
  scanner.Split(bufio.ScanWords)
  count := 0
  for scanner.Scan() {
    count++
  }
  if err := scanner.Err(); err != nil {
    fmt.Println(err)
    success <- false
    return
  }
  fmt.Println(name, count)
  success <- true
}

func main() {
  files := []string{"data1.txt", "data2.txt"}
  s := make(chan bool)
  for _, f := range files {
    go countWords(f, s)
  }
  for i := 0; i < len(files); i++ {
    <- s
  }
}
```

## Synchronization
There are several ways to use channels to synchronize goroutines
- using a channel to send results
- using a channel to signal completion
- `close()`

### The `close` Function
The close function is a builtin function that closes a channel
to indicate that nothing else will be sent over the channel.
Sending over a closed channel causes a runtime panic.
Receiving from a closed channel will give you a zero-value for the
channel's type.

#### Signaling End of Output
```go
func fib(n int, c chan int) {
    x, y := 0, 1
    for i := 0; i < n; i++ {
        c <- x
        x, y := y, x + y
    }
    close(c)
}

func main() {
    c := make(chan int)
    go fib(10, c)
    for i := range c {
        fmt.Println(i)
    }
}
```

##### Note: closed channels can be reopened with `make(chan [type])`

#### Signaling End of Input
```go
type Coord struct {
    x, y float64
}

func printDistance(coords chan Coord, done chan bool) {
    for c := range coords {
        fmt.Println("Distance", math.Hypot(c.x, c.y))
    }
    done <- true
}

func main() {
    c, d := make(chan Coord), make(chan bool)
    for i := 0; i < 3; i++ {
        go printDistance(c, d)
    }
    for i := 0; i < 10; i++ {
        c <- Coord{rand.Float64(), rand.Float64()}
    }
    close(c)
    for i := 0, i < 3; i++ {
        <- d
    }
}
```

## Select Statement
A select statement waits on multiple channel operations, and runs the first
communication operation that is ready. If multiple are ready, one is run
at random.

```go
func main() {
    c := make(chan int)
    quit := time.After(5 * time.Millisecond)
    go func() {
        for i := 0; ; i++ {
            c <- i
        }
    }()
    for {
        select {
        case val := <- c:
            fmt.Println(val)
        case <- quit:
            fmt.Println("quit")
            return
        }
    }
}
```
