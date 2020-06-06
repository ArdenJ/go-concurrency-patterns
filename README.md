# Golang Concurrency Patterns

**This is a summary of the mental models and concurrency patterns [discussed by Arne Clause at the Golang UK Conference in 2017](https://www.youtube.com/watch?v=rDRa23k70CU).**

Concurrency is not Parallelism. Currency is about designing a program as a collection of independent processes that are made to run in parallel in such a way that the outcome is predictable.

## How does go do 

Communicating sequential processes 
- Each process is built for sequential execution 
- Data is *communicated* between processes via channels. 
**NO SHARED STATE** 
  - In theory this means that you don't get deadlocks and you don't have race conditions
  - If data needs to be shared a copy is communicated.
- Scale by adding  more of the same 

## What do we have for doing concurrency? 
- Go routines
- channels
- select
- sync package (mutexs, condition variable)

### CHANNELS
Channels can be thought of as a chain of buckets with one horrible monster creature passing it to the next in order to put out a fire.

There are three components: **senders**, **buffer** (optional), **receiver**

#### When would this model block? 
💤 If a sender doesn't have a receiver 
💤 If a sender fills a buffer but doesn't have a receiver 
💤 If a receiver doesn't have a sender
💤 If a buffer doesn't have a sender to get data for a receiver 

(*N.B. - buffers are dumb they are just space to fill*)

##### unbuffered examples
```go
unbuffered := make(chan int)

// 1) we want to get something from the channel but block 
// because there is no data (sender)
a := <-unbuffered 

// 2) we try to send something to the channel but is blocks because the channel has no receiver
unbuffered <-1

// 3) we need a sender and a receiver so we start a go routinen and send. This blocks until both elements reach the channel (it's a simple form of synchronising pattern). (note that if we were to wrap the sender in a go routine then the program would exit before doing anything (as neither would reach the channel - blocking allows us to synchronise))
go func() { <-unbuffered }()
unbuffered <-1
```

##### buffered examples
```go 
buffered := make(chan int, 1)

// 4) this blocks - still no data
a := <-buffered 

// 5) this is fine for the first call as there is a buffer with one space. A second call will block
buffered <-1
```

#### blocking breaks concurrency 
Blocking can lead to deadlocks. It can also prevent scaling - if the program bottlenecks or blocks at a certain point then no amount of additional goroutines is going to help speed things up

#### closing channels
closing always generates a message. A message bubbles through the entire channel (it should come from the sender!!!) and is received closing the channel.

If you try to send more down a closed channel the program will *panic*!

Closing twice will panic as you will end up sending a message down an already closed channel 

```go 
c := make(chan int)
close(c)

fmt.Println(<-c) // receive and print from the channel

// what gets printed?? it must do something as closing generates a message: we get "0, false" this is a zero value of channel type (0), and false to indicate _no more datas_
```

### SELECT
Select is like a switch statement for channel operations. The order of its cases doesn't matter and it will select the first non-blocking case to send/receive on. It also has a default case which come in handy for creating a potentially non-blocking channel

```go 
// this is a little unrealistic - if there is data and more to come we select the first case else we send the 0 message but tell the reciever that there is no data at present but there is more preventing it from blocking. 
func TryReceive(c<- chan int) (data int, more, ok bool) {
  select {
    case data, more = <-c:
    return data, more, true

    default: // the process when c is blocing
    return 0, true, false
  }
}

// THis channel is blocking immediately and is blocking until the duration is over then it will send the time channel was unblocked. 
func TryReceiveWithTimeOut(c<- chan int, duration time.Duration) (data int, more, ok bool) {
  select {
    case data, more = <-c: 
    return data, more, true

    case <-time.After(duration): // time.After() returns a channel
    return 0, true, false
  }
}
```

#### Select is about routing data 
You can use select to really shape your dataflow. 

fanning out data, funnelling them in or receiving multiple requests and turning the out to the appropriate data.

##### Fan-out 
A little like a load balancer 
```go 
func Fanout(In<- chan int, OutA, OutB chan int) {
  for data := range In { // receive data until closed
    select { // Send on the first non-blocing channel
      case OutA<- data
      case OutB<- data
    }
  }
}
```

##### Turnout

Note the !more: in select if you have a closed channel - it will always be selected. You will never get to the other channels

```go 
func Turnout(InA, InB <- chan int, OutA, OutB chan int) {
  // variable declaration omitted!!!
  for {
    select { // receive from first non-blocking channel
      case data, more = <- InA:
      case data, more = <- InB:
    }
    if !more {
      return 
    }
    selct { // send on first non-blocking
      case OutA <- data:
      case OutB <- data:
    }
  }
} 
```
To get around the issue in the about we can add a QUIT channel which only exists to be closed

```go 
func Turnout(Quit <-chan int, InA, InB, OutA, OutB chan int) {
  // variable declaration omitted again 

  for {
    select {
      case data = <-InA:
      case data = <-InB:

      case <-Quit: // close generates a message
        close(InA) // THIS IS AN ANTI PATTERN: 
        close(InB) //we are closing on the reciever but it could be argued that quit is acting as a delegator

        Fanout(InA, OutA, OutB) // flush out the remaining data
        Fanout(InB, OutA, OutB)
        return
    }
  }
}
```

**90% of use cases can be handled with CSP, Channels, and Select**

You can create deadlocks with channels - we can make them all unblocking however sometime we want to block a channel e.g. to synchronise some part of the programme

channels pass around copies to communicate which can impact performance at scale

passing pointers to channels can create race conditions 

**how are channels supposed to deal with naturally shared structures such as caches or registries?**

The answer is in the sync package...

**Mutexes are not optimal**
- They don't stop incoming data - the longer they're occupied the longer the queue gets
- Read/write locked mutexes can only *reduce* the problem 
  - One channel locks the shared structure causing the next to block ( this is the bank deposit example )
  - Good for when you have a lot more reads than writes
- Using multiple mutexes will cause deadlocks sooner or later ((HEY ARDEN THIS IS WHERE A FSM COULD BE GUD))

### Three shades of code 
- **Blocking** = your program can get locked up (for a period of time)
  - e.g. when using a mutex and a channel is waiting to write on a lock
- **Lock-free** = at least one part of the program is *always* making progress
- **Wait-free** = All parts of your program are always making progress
  - This doesn't... really exist 

### SYNC
#### Atomic operations in the sync package 
**sync.atomic**

The package is small and only includes the following functions: 
**Store, Load, Add, Swap, CompareAndSwap**

These are mapped to thread-safe CPU instructions! Meaning they don't require mutexes! This functionality comes at a trade-off, however, and they are roughly 10-60x slower than non-atomic operations (i.e. don't just use them everywhere).

#### Atomic Patterns
##### Spinning CompareAndSwap
- You need a **state** and a *free* constant
- Use CAS in a loop 
  - if state is *not free* you try again until it is 
  - if state is *free* set it to something else 
- If you managed to change the state the you *own* it. until you set it back 

**it's basically a lock** - the sync mutex uses the same implementation

```go 
type SpinLock struct {
  state *int32
}

const free = int32(0)

func (l *SpinLock) lock() {
  for !atomic.CompareAndSwapInt32(l.state, free, 42) { // 42 or any non-0 value
    runtime.Gosched() // nudge the scheduler
  }
}

func (l *SpinLock) Unlock() {
  atomic.StoreInt32(l.state, free)
}
```
A normal mutex would have blocked but instead anything else that tries to do something is just forever spinning 

N.B. If it is atomic **STAY ATOMIC**

##### Ticket Storage 
- We need an **indexed data structure**, a **ticket** and a **done** variable
- A function draws a new ticket by adding 1 to the ticket 
- Every ticket number is *unique* as we *never* decrement
- Treat the **ticket as an index** to store your data
- Increase done to extend the "ready to read" range (the write itself is not atomic)

```go 
type TicketStore struct {
  ticket *uint64
  done *uint64
  slots []string // assumed to be infinite for simplicity (normally this would be something like a ring buffer)
}

func (ts *TicketStore) Put(s string) {
  t := atomic.AddUint64(ts.ticket, 1) -1 //draw a ticket (-1 because we get the value after the operation)
  slots[t] = s //store your data

  // this stops us for being wait-free here (it waits until the done variable has the state of our ticket before moving on. debug example below shows how not waiting will result inn a race condition)
  for !atomic.CompareAndSwapUint64(ts.done, t, t+1) { //increase done
    runtime.Gosched()
  }
}

// this is effectively wait free 
// it will always return the portion of the slice that is done
func (ts *TicketStore) GetDone() []string {
  return ts.slots[:stomic.loadUint64(ts.done)+1] // read up to done
}
```

##### DEBUGGING ATOMIC CODE 
**with the "instruction pointer game"** 

- Pull up **two windows** (= two go routines with the same code)
- You have **one instruction pointer** that iterates through your code 
- you may **switch** windows **at any instruction** 
- **Watch** variables for race conditions 

```go 
func(ts *TicketStore) Put(s string) {
  // GET TICKET #1
  ticket := atmic.AddUint64(ts.next, 1) -1
  slots[ticket] = s

  atomic.AddUint64(ts.done, 1)
}
```
⬇️⬇️⬇️⬇️⬇️
```go 
func(ts *TicketStore) Put(s string) {
  // GET TICKET #2
  ticket := atmic.AddUint64(ts.next, 1) -1
  // write to the slot
  slots[ticket] = s

// incrementing done = #1 ⚠️ RACE CONDITION ⚠️
  atomic.AddUint64(ts.done, 1)
}
```

## Guidelines for non-blocking code
- Do not switch between atomic and non-atomic functions
- target and exploit situations which enforce uniqueness 
  - e.g. look for moments where yo may be the only one with access to that variable, or the only one changing that value
- Avoid changing two things at the same time 
  - Sometimes you can exploit bit operation 
    - This is cool: you can use to 32bit values and one 64bit int 
  - Sometimes intelligent ordering will help
  - But sometimes it may just not be possible 

## Concurrency in Practice
- Avoid blocking; avoid race conditions
- Use channels to avoid shared state
  - use select to manage channels
- When channels won't work 
  - Try to use tools from the sync package first 
    - use a mutex
  - in simple cases or when *really* needed: try lockless atomic code