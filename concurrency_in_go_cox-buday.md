# Chapter 1: An Introduction to Concurrency

## Why is Concurrency Hard?

### Atomicity
* atomic: within the context in which something is operating, it is indivisible or uninterruptible.

### Deadlocks, Livelocks and Starvation
* deadlock
* The Coffman Conditions must be present for deadlocks to arise:
    * Mutual exclusion: a concurrent process holds exclusive rights to a resource at any one time.
    * Wait-for condition: a concurrent process must simultaneously hold a resource and be waiting for an additional resource.
    * No preemption: a resource held by a concurrent process can only be released by that process.
    * Circular wait: a concurrent process (P1) must be waiting on a chain of other concurrent processes (P2), which are in turn waiting on P1.
* livelock
* livelocks are programs that are actively performing concurrent operations, but these operations do nothing to move the state of the program forward.
* Commonly a product of attempting to prevent a deadlock without coordination.
* starvation
* Any situation where a concurrent process cannot get all the resources it needs to perform work.


# Chapter 2: Modeling Your Code: Communicating Sequential Processes

## The Difference Between Concurrency and Parallelism
* concurrency is a property of the code; parallelism is a property of the running program. For example, a program may be written in a concurrent manner, but if the machine has only one core, it will not be run in parallel.


# Chapter 3: Go's Concurrency Building Blocks

## goroutines
* green threads: threads managed by a language's runtime.
* goroutines are not threads or green threads, but coroutines.
* coroutine: concurrent subroutines (functions, closures or methods) that are non-preemptive. I.e. they cannot be interrupted.
    * coroutines have multiple points throughout which allow for suspension or reentry.
* goroutines don't define their own suspension or reentry points. Go's runtime observes their behaviour and automatically suspends them when they block and resumes them when they become unblocked.
* Go's mechanism for hosting goroutines is an implementation of an M:N scheduler, which means it maps M green threads to N OS threads.
* When there are more goroutines than available green threads, the scheduler handles their distribution across the available threads. 
* Go follows a model of concurrency called the fork-join model. At any point in the program, it can split of a child branch of execution to be run concurrently with its parent.
    * Where a child rejoins is called a __join point__.
* goroutines execute within the same address space they were created in.
* If a variable captured by a goroutine closure falls out of scope, the Go runtime transfers the memory to the heap so that the goroutine can continue to access it.
* The garbage collector does nothing to collect goroutines that have been abandoned (goroutine leak).
* context switching: when something hosting a concurrent process must save its state to switch to running a different concurrent process.
    * An OS thread must save things like register values, lookup tables and memory maps.
    * Under a software-defined scheduler., the Go runtime can be more selective in what is persisted, how it's persisted, and when the persisting occurs.

## The `sync` Package

### `Cond`
* A rendezvous point for goroutines waiting for or announcing the occurrence of an event.
* 
  ```Go
  c := sync.NewCond(&sync.Mutex{})
  c.L.Lock()
  for conditionTrue() == false {
    c.Wait()
  }
  c.L.Unlock()
  ```
* `Wait` automatically calls `Unlock` when it is triggered and `Lock` when it is exited.
* It's important to check the condition in a loop, because a signal on the `Cond` doesn't necessarily mean that what you've been waiting for has occurred, only that __something__ has occurred.
* The runtime maintins a FIFO list of goroutines waiting to be signaled.
* `Cond.Signal` finds the goroutine that's been waiting the longest and notifies that.
* `Cond.Broadcast` sends a signal to all goroutines that are waiting.
* E.g. could be used to broadcast when a button has been clicked to multiple handlers.

### `Once`
* `sync.Once` only counts the number of times `Do` is called, not how many times unique functions passed into `Do` are called.

### `Pool`
* The pool pattern is a way to create and make available a fixed number of things for use.
* Commonly used to constrain the creation of things that are expensive (e.g. database connections.)
* Used for warming a cache of pre-allocated objects for operations that much run as quickly as possible.
* Best used when you have concurrent processes that require objects, but dispose of them very rapidly after instantiation, or when construction of these objects could negatively impact memory.
* If  the code that utilizes the `Pool` requires things that are not roughly homogeneous, you may spend more time converting what you've retrieved from that `Pool` than it would have taken to instantiate it in the first place.

## channels
* buffered channels are an in-memory FIFO queue for concurrent processes to communicate over.
* If a buffered channel is empty and has a receiver, the buffer will be bypassed and the value will be passed directly from the sender to the receiver.
* If a goroutine making writes to a channel has knowledge of how many writes it will make, it can be useful to create a buffered channel whose capacity is the number of writes to be made, and then make those writes as quickly as possible.
* Reading from and writing to a `nil` channel will block.
* Closing a `nil` channel will panic.
* Writing to or closing a `nil` channel will panic.
* channels require proper __ownership__:
* The goroutine that owns the channel should:
    * Instantiate the channel.
    * Perform writes, or pass ownership to another goroutine.
    * Close the channel.
    * Encapsulate the these things and expose them via a reader channel.
* A consumer of a channel needs to:
    * Know when a channel is closed.
    * Resposibly handle blocking for any reason.

## The `select` Statement
* `select {}` will block forever.

## The GOMAXPROCS Lever
* Controls the number of OS threads that will host "work queues".


# Chapter 4: Concurrency Patterns in Go

## confinement
* Ensuring information is only ever available from one concurrent process.
* Two kinds of confinement: ad-hoc and lexical.
* Ad hoc confinement is achieved by convention.
* Lexical confinement involves using lexical scope to expose only the correct data and concurrency primitives for multiple concurrent processes to use.

## Preventing goroutine leaks
* Establish a signal between the parent goroutine and its children that allows the parent to signal cancellation to its children, usually a read-only channel named done.
* As a convention, `done` is passed as the first parameter.
*
  ```Go
  doWork := func(
    done <- chan interface{}
    strings <- chan string,
  ) <- chan interface{} {
    terminated := make(chan interface{})
    go func() {
      defer fmt.Println("doWork exited.")
      defer close(terminated)
      for {
        select {
        case s := <-strings:
          // Do something
        case <- done:
          return
        }
      }
    }()
    return terminated
  }
  ```
    * If a goroutine is responsible for creating a goroutine, it is also responsible for ensuring it can stop the goroutine.
  ## The or-channel
    * Used to combine one or more `done` channels into a single `done` channel that closes if any of its parent components close.
    *
```Go
or := func(channels ...<- chan interface{}) <-chan interface {
  switch len(channels) {
  case 0:
    return nil
  case 1:
    return channels[0]  
  }
    
  orDone := make(chan interface{})
  go func() {
    defer close(orDone)
    
    switch len(channels) {
    case 2:
      select {
      case <-channels[0]:
      case <-channels[1]:
      }
    default:
      select {
      case <-channels[0]:
      case <-channels[1]:
      case <-channels[2]:
      case <-or(append(channels[3:], orDone)):
       }  
    }
  }()
  return orDone
}```
    * Useful at the intersection of modules in a system, where you tend to have multiple conditions for cancelling trees of goroutines through your call stack.
  ## Error Handling
    * concurrent processes should send their errors to another part of the system that has complete information about the state of your program, and can make a more informed decision about what to do.
    * ```rust
type Result struct {
  Error error
  Response *http.Response
}

checkStatus := func(done <-chan interface{}, urls ...string) <-chan Result {
  results := make(chan Result)
  go func {
    defer close(results)
    
    for _, url := range urls {
      var result Result
      resp, err := http.Get(url)
      result = Result{Error: err, Response: resp}
      select {
      case <-done:
        return
      case results <- result:
      }
    }
  }()
  return results
}```
  ## pipelines
    * pipelines are a powerful abstraction when your program needs to process streams, or batches of data.
    * The properties of pipeline stages:
      * A stage consumes and returns the same type.
      * A stage must be reified by the language so that it may be passed around.
        * Reification: a language exposes a concept to the developers so that they can work with it directly.
    * batch processing pipeline stages operate on chunks of data like slices.
    * stream processing pipeline stages receive and emit one element at a time.
    * At the beginning of a pipeline, we must convert discrete values into a channel. There are two points in this process that must be preemptible:
      * Creation of the discrete value (if it is not practically instantaneous).
      * Sending the discrete value on its channel.
    * The final stage of a pipeline is assured preemptibility because the channel it's ranging over will be closed when an earlier stage is preempted, thus breaking the range. I.e. the final stage is preemptible because the stream(s) it relies on are preemtible.
    * If a stage is blocked on retrieving a value from the incoming channel, it will become unblocked when that channel is closed. We know by induction that the channel will be closed because it is either a stage written like the stage we are within, or the beginning of the pipeline that we have written to be preemptible. Thus, our entire pipeline is always preemptible by closing the `done` channel.
  ## Fan-Out, Fan-In
    * Fan-out is a term to describe the processes of starting multiple goroutines to handle input from the pipeline.
    * Fan-in describes multiplexing or joining together multiple streams of data into a single stream.
    * Consider fanning out a stage if both of the following apply:
      * It doesn't rely on values that it had previously calculated.
      * It takes a long time to run.
      *
```Go
numFinders := runtime.NumCPU()
finders := make([]<-chan int, numFinders)
for i := 0; i < numFinders; i++ {
  finders[i] = primeFinder(done, randIntStream)
}```
    * Fanning in:
      *
```Go
fanIn := func(
  done <-chan interface{},
  channels ...<-chan interface{},
) <-chan interface{} {
  var wg sync.WaitGroup
  multiplexedStream := make(chan interface{})
  
  mutliplex := func(c <-chan interface{}) {
  defer wg.Done()
  for i := range c {
    select {
    case <-done:
    return
    case multiplexedStream <- i:
    }
  }
  }

  // Select from all the channels
  wg.Add(len(channels))
  for _, c := range channels {
  go multiplex(c)
  }

  // Wait for all the reads to complete
  go func() {
  wg.Wait()
  close(multiplexedStream)
  }()

  return multiplexedStream
}```
      * Fanning in involved create the multiplexed channel consumers will read from, and then spinning up one goroutine for each incoming channel, and one goroutine to close the multiplexed channel when the incoming channels have all been closed.
    * A naive implementation of the fan-in, fan-out algorithm works only if the order in which results arrive is unimportant.
  ## The or-done Channel
    * When working with channels from disparate parts of your system, you can't make any assertions about how a channel will behave when code you're working with is canceled via its `done` channel. You don't know if that fact that __your__ goroutine has been canceled means the channel you're reading from will have been canceled.
      * We need to wrap our read from the channel with a `select` statement that also select from a `done` channel.
      * Allows you to exit a select when an upstream goroutine that you don't have control over has been closed, not just when the goroutine you have control over is cancelled.
    *
```Go
orDone := func(done, c <-chan interface{}) <-chan interface{} {
  valStream := make(chan interface{})
  go func() {
  defer close(valStream)
  
  for {
    select {
    case <-done:
    return
    case v, ok := <-c:
    if !ok {
      return
    }
    select {
    case valStream <- v:
    case <-done:
    }
    }
  }
  }
}```
  ## The tee-channel
    * Used to split incoming values from a channel so you can send them to two separate destinations.
    *
```Go
tee := func(done, in <-chan interface{}) (_, _ <-chan interface) {
  out1 := make(chan interface{})
  out2 := make(chan interface{})
  go func() {
  defer close(out1)
  defer close(out2)
  
  for val := range orDone(done, in) {
    var out1, out2 = out1, out2
    for i := 0; i < 2; i++ {
  select {
    case <- done:
    case out1 <- val:
      out1 = nil
    case out2 <- val:
      out2 = nil
    }
    }
  }
  }()
  return out1, out2
}```
  ## The bridge-channel
    * A bridge destructures a channel of channels into a single channel.
    *
```Go
bridge := func(
  done <-chan interface{},
  chanStream <-chan <-chan interface{},
) <-chan interface{} {
  valStream := make(chan interface{})
  go func() {
  defer close(valStream)
  
  for {
    var stream <-chan interface{}
    select {
    case maybeStream, ok := <-chanStream:
    if ok == false {
      return // chanStream has been closed
    }
    stream = maybeStream
    case <-done:
    return
    }
    
    for val := range orDone(done, stream) {
    select {
    case valStream <- val:
    case <-done:
    }
    }
  }
  }()
  return valStream
}```
  ## queueing
    * The process of accepting work for your pipeline evening though it's not yet ready for more.
    * queueing is one of the last techniques to employ when optimising a program. Adding queueing prematurely can hide synchronisation issues such as deadlock and livelock.
    * queuing doesn't reduce the runtime of a pipeline, but reduces the blocking time of particular stages.
    * The true utility of a queue is to decouple stages so that the runtime of one stage has no impact on the runtime of another.
    * The only situations where queueing can increase the overall performance of your system:
      * If batch processing requests in a stage saves time.
        * Includes if your algorithm can be optimised by supporting lookbehinds, or ordering.
      * If delays in a stage produce a feedback loop in the wider system.
    * Anytime performing an operation requires an overhead, chunking may increase system performance. E.g. opening database transactions, calculating message checksums, and allocating contiguous space.
    * In a situation where a delay in a stage causes more input into the pipeline, this can result in a negative feedback loop: the rate at which upstream stages or systems submit new requests is somehow linked to how efficient the pipeline is.
      * E.g. too many people trying to access a server at once. The more people try to access it, the slower it becomes, and the fewer new entrants get processed.
      * By introducing a queue at the entrance to the pipleine, you can break the feedback loop at the cost of creating lag for requests.
    * queueing should be implemented either:
      * At the entrance to a pipeline.
      * In stages where batch processing will leading to higher efficiency.
    * Little's Law predicts the throughput of a pipeline:
      * $$L=λW$$
      * $$L$$: the average number of units in the system.
      * $$λ$$: the average arrival rate of units.
      * $$W$$: the average time a unit spends in the system.
      * Only applies to stable systems, where the rate of ingress is equal to the rate of egress.
      * If we want to decrease $$W$$, the average time a unit spends in the system, by a factor of $$n$$, we have only one option: to decrease the average number of units in the system, and we can only do that by increasing the rate of egress.
      * If we add queues, we increase $$L$$, which either increases the arrival rate of units ($$nL = nλ * W$$) or increased the average time a unit spends in the system ($$nL = λ * nW$$).
      * This proves that queueing cannot help decrease the amount of time spent in a system.
      * Since we're observing the pipeline as a whole, reducing $$W$$ by a factor of $$n$$ is distributed throughout all stages of our pipeline. Little's Law should really be defined as:
        * $$L = λ∑_iW_i$$
      * As you increase queue size, your work takes longer to make it through the system. You trade system utilisation for lag.
    * If your pipeline panics, you'll lose all requests in your queue.
      * To mitigate this, don't use queues or move to a persistent queue.
  ## The context Package
    * When using context, each function downstream from your top-level call takes a `Context` as its first argument.
    * context serves two purposes:
      * To provide an API for cancelling branches of your call graph.
      * To provide a data-bag for transporting request-scoped data through your call-graph.
    * Cancellation has three aspects:
      * A goroutine's parent may want to cancel it.
      * A goroutine may want to cancel its children.
      * Any blocking operations within a goroutine need to be preemptible so that it may be cancelled.
    * context structs are immutable.
    * There's nothing that allows the function accepting the context to cancel it. This protects functions up the call stack from children cancelling the `Context`.
    * `WithCancel` returns a new context that closes its `done` channel when the returned `cancel` function is called.
    * `WithDeadline` returns a new context that closes its `done` channel when the machine's clock advances past the given `deadline`.
    * `WithDeadline` returns a new context that closes its `done` channel after the given `timeout` duration.
    * Instances of `Context.context` may look equivalent, but internally they change every stack frame.
      * It's important to only pass instances of context to functions, never pointers.
    * To implement a `done` channel with context:
      *
```Go
func main() {
  var wg sync.WaitGroup
  ctx, cancel := context.WithCancel(context.Background())
  defer cancel()
  
  wg.Add(1)
  go func() {
  defer wg.Done()
  
  if err := printGreeting(ctx); err != nil {
    fmt.Printf("cannot print greeting: %v\n", err)
    cancel()
  }
  }()
  
  wg.Add(1)
  go func() {
  defer wg.Done()
  
  if err := printFarewell(ctx); err != nil {
    fmt.Printf("cannot print farewell: %v\n", err)
  }
  }()
  
  wg.Wait()
}

func printGreeting(ctx context.Context) error {
  greeting, err := genGreeting(ctx)
  if err != nil {
  return err
  }
  fmt.Printf("%s world!\n", greeting)
  return nil
}

func printFarewell(ctx context.Context) error {
  farewell, err := genFarewell(ctx)
  if err != nil {
  return err
  }
  fmt.Printf("%s world!\n", farewell)
  return nil
}

func genGreeting(ctx context.Context) (string, error) {
  ctx, cancel := context.WithTimeout(ctx, 1*time.Second)
  defer cancel()
  
  switch locale, err := local(ctx); {
  case err != nil:
  return "", err
  case locale == "EN/US":
  return "hello", nil
  }
  return "", fmt.Errorf("unsupported locale")
}
  
func genFarewell(ctx context.Context) (string, error) {
  switch locale, err := locale(ctx); {
  case err != nil:
  return "", err
  case locale == "EN/US":
  return "goodbye", nil
  }
  return "", fmt.Errorf("unsupported locale")
}
  
func locale(ctx context.Context) (string, error) {
  select {
  case <-ctx.Done():
  return "", ctx.Err()
  case time.After(1*time.Minute):
  }
  return "EN/US", nil
}```
    * If we know how long part of our pipeline takes to run, we can check to see if we were given a deadline, and if so, whether we'll meet it:
      *
```Go
func locale(ctx context.Context) (string, error) {
  if deadline, ok := ctx.Deadline(); ok {
  if deadline.Sub(time.Now().Add(1*time.Minute)) <= 0 {
    return "", context.DeadlineExceeded
  }
  }
  
  select {
  case <-ctx.Done():
  return "", ctx.Err()
  case <- time.After(1*time.Minute):
  }
  return "EN/US", nil
}```
    * Often when a function creates a goroutine and a context, it's starting a process that will service requests, and functions further down the stack may need information about the request.
      *
```Go
func ProcessRequest(userID, authToken string) {
  ctx := context.WithValue(context.Background(), "userID", userID)
  ctx = context.WithValue(ctx, "authToken", authToken)
  HandleRespose(ctx)
}```
      * The key must be comparable.
      * Values must be safe to access from multiple goroutines.
      * It's recommended to define a custom key-type in your package, preventing key collisions between different packages within a single context.
    * Use context values only for request-scoped data that transits processes and API boundaries, not for passing optional parameters to functions.
      * The data should transit process or API boundaries.
      * The data should be immutable.
      * The data should trend towards simple types.
      * The data should be data, not types with methods.
      * The data should help decorate operations, not drive them.
        * I.e. your algorithm should not behave differently based on what is or isn't included in its `Context`.
# Chapter 5: Concurrency at Scale
  ## Error Propagation
    * Errors should always relay critical pieces of information:
      * What happened.
        * Likely to be generated implicitly by whatever generated the errors.
      * When and where it occurred.
        * Full stack trace.
        * Information about the context it's running in. E.g. on a distributed system, what machine it occurred on.
        * The time on the machine the error was instantiated on, in UTC.
      * A friendly user-facing error message.
        * The message should only contain abbreviated and relevant information from the preceding points.
        * Human-centric, indicates whether error is transitory, and should be about one line of text.
      * How the user can get more information.
        * Should contain an ID that can be cross-referenced with a corresponding log that displays the full info
    * All errors fall into one of two categories:
      * Bugs.
      * Known edge cases (e.g. broken network connections, failed disk writes).
    * Bugs are errors that you have not customized to your system, or "raw" errors
    * At the boundaries of each component, all incoming errors must be wrapped in a well-formed error for the component our code is within.
      * Any error that escapes our module with our module's error type can be considered malformed, and a bug.
      * It's only necessary to wrap errors at your own module boundaries (public functions/methods), or when your code can add valuable context.
    * When malformed errors, or bugs, are propagated up to the user, we should also log the error, but then display a friendly message to the user stating something unexpected has happened.
    *
```Go
type MyError struct {
  Cause error
  Message string
  StackTrace string
  Misc map[string]interface{}
}

func wrapError(err error, messagef string, msgArgs ...interface{}) MyError {
  return MyError{
  Cause: err,
  Message: fmt.Sprintf(messagef, msgArgs...),
  StackTrace: string(debug.Stack()),
  Misc: make(map[string]interface{}),
  }
}

func (err MyError) Error() string {
  return err.Message
}```
    * In a specific module, `MyError` would be used thus:
      *
```Go
// "lowlevel" module

type LowLevelErr struct {
  error
}

func isGloballyExec(path string) (bool, error) {
  info, err := os.Stat(path)
  if err != nil {
  return false, LowLevelErr{(wrapError(err, err.Error()))}
  }
  ...
}```
    * At the top level, the error is handled thus:
      *
```Go
func main() {
  log.SetOutput(os.Stdout)
  log.SetFlags(log.Ltime|log.LUTC)
  
  err := runJob("1")
  if err != nil {
  msg := "There was an unexpected issue; please report this as a bug."
  if _, ok := err.(IntermediateErr); ok {
    msg = err.Error()
  }
  handleError(1, err, msg)
  }
}```
  ## Timeouts and Cancellation
    * There are several reasons concurrent processes might need to support timeouts:
      * System saturation. We may want requests at the edge of our system to time out rather than take a long time to process if our system is saturated. Time out if:
        * The request is unlikely to be repeated. If it can be repeated, the system will develop an overhead from accepting and timing out requests which can lead to a death-spiral.
        * You don't have the resources to store requests (e.g. memory for in-memory queues, disk space for persisted queues).
        * If the need for the request or the data it's sending will go stale.
          * If the window is known beforehand, it makes sense to pass our concurrent process a `context.Context` created with `context.WithDeadline` or `context.WithTimeout`
          * If the window is not known beforehand, we'd want the parent of the concurrent process to be able to cancel it when the need for the request is no longer present. `context.WithCancel` is ideal for this purpose.
    * There a a number of reasons why a concurrent process might be cancelled:
      * Timeouts. A timeout is an implicit cancellation.
      * User intervention.
      * Parent cancellation. If a parent of a concurrent process stops for any reason, its children must be cancelled.
      * Replicated requests. We may wish to send data to multiple concurrent processes in an attempt to get a faster response from one of them. When the first one comes back, we want to cancel the others.
    * When designing a cancellable goroutine, you should aim for all nonpreemptible atomic operations to complete in less than the time period you've deemed acceptable. I.e. the goroutine should be preemptible between every distinct step of its operation.
    * To avoid sending duplicate messages:
      * Make it vanishingly unlikely that a parent goroutine will send a cancellation signal after a child goroutine has already reported a result.
        * Requires bidirectional communication between stages (see heartbeats).
      * Accept either the first or last result reported.
        * If the algorithm allows it, or the concurrent process is idempotent, allow for the possibility of duplicate messages and choose whether to accept the first or last message you receive.
      * Poll the parent goroutine for permission.
        * Use a bidirectional channel with the parent to explicitly request permission to send your message downstream.
        * Safer than heartbeats, but more complex.
  ## heartbeats
    * heartbeats are a way for concurrent processes to signal life to outside parties.
    * Allow insights into a system and and make testing the system deterministic when it might otherwise not be.
    * Two types of hearbeats:
      * Occurring on a time interval:
        *
```Go
doWork := func(
  done <-chan interface{},
  pulseInterval time.Duration,
) (<-chan interface{}, <-chan time.Time) {
  heartbeat := make(chan interface{})
  results := make(chan time.Time)
  go func() {
  defer close(heartbeat)
  defer close(results)
  
  pulse := time.Tick(pulseInterval)
  workGen := time.Tick(2*pulseInterval)
  
  sendPulse := func() {
    select {
    case heartbeat <- struct{}{}:
    default:
    }
  }

  sendResult := func(r time.Time) {
    for {
    select {
    case <-done:
      return
    case <-pulse:
      sendPulse()
    case results <- r:
      return
    }
    }
  }

  for {
    select {
    case <-done:
    return
    case <-pulse:
    sendPulse()
    case r := <-workGen:
    sendResult(r)
    }
  }
  }()
  return heartbeat, results
}```
      * Occurring at the beginning of a unit of work:
        *
```Go
doWork := func(done <-chan interface{}) (<-chan interface{}, <-chan int) {
  heartbeatStream := make(chan interface{}, 1)
  workStream := make(chan int)
  go func() {
  defer close(heartbeatStream)
  defer close(workStream)
  
  for i := 0; i < 10; i++ {
    select {
    case heartbeatStream <- struct{}{}:
    default:  
    }
  
    select {
    case <-done:
    return
    case workStream <- rand.Intn(10):
    }
  }
  }()
  return heartbeatStream, workStream
}```
          * `heartbeat` is buffered to ensure that at least one pulse always goes out even if nothing is listening in time for the send to occur.
    * heartbeats can be used in tests to check that a goroutine has started doing its work:
      *
```Go
func DoWork(
  done <-chan interface{},
  nums ...int,
) (<-chan interface{}, <-chan int) {
  heartbeat := make(chan interface{}, 1)
  intStream := make(chan int)
  go func() {
  defer close(heartbeat)
  defer close(intStream)
  
  time.Sleep(2*time.Second)
  
  for _, n := range nums {
    select {
    case heartbeat <- struct{}{}:
    default:
    }
  
    select {
    case <-done:
    return
    case intStream <- n:
    }
  }
  }()

  return heartbeat, intStream
}

func TestDoWork_GeneratesAllNumbers(t *testing.T) {
  done := make(chan interface{})
  defer close(done)

  intSlice := []int{0, 1, 2, 3, 5}
  heartbeat, results := DoWork(done, intSlice...)

  <-heartbeat
  
  i := 0
  for r := range results {
  if expect := intSlice[i]; r != expected {
    t.Errorf("index %v: expected %v, but received %v" i, expected, r)
  }
  i++
  }
}```
        * This structure ensures that the test only proceeds when the goroutine is ready to send results.
        * The only risk is of one iteration taking an inordinate amount of time. If that's a concern, use the safer interval-based heartbeats:
          *
```Go
func DoWork(
  done <-chan interface{},
  pulseInterval time.Duration,
  nums ...int,
) (<-chan interface{}, <-chan int) {
  heartbeat := make(chan interface{}, 1)
  intStream := make(chan int)
  go func {
  defer close(heartbeat)
  defer close(intStream)
  
  time.Sleep(2*time.Second)
  
  pulse := time.Tick(pulseInterval)
  numLoop:
  for _, n := range nums {
    for {
    select {
    case <-done:
      return
    case <-pulse:
      select {
      case heartbeat <- struct{}{}:
      default:
      }
    case intStream <- n:
      continue numLoop
    }
    }
  }
  }()

  return heartbeat, intStream
}

func TestDoWork_GeneratesAllNumbers(t *testing.T) {
  done := make(chan interface{})
  defer close(done)

  intSlice := []int{0, 1, 2, 3, 5}
  const timeout = 2*time.Second
  heartbeat, results := DoWork(done, timeout/2, intSlice...)

  <-heartbeat

  i := 0
  for {
  select {
  case r, ok := <-results:
    if ok == false {
    return  
    } else if expected := intSlice[i]; r != expected {
    t.Errorf(
      "index %v: expected %v, but received %v",
      i,
      expected,
      r,
    )
    }
    i++
  case <-heartbeat:
  case <-time.After(timeout):
    t.Fatal("test timed out")  
  }
  }
}```
            * The logic of this version of the test is less clear. If you're reasonably sure the goroutine's loop won't stop executing once it's started, it's recommended to block only on the first heartbeat and then falling into a simple range statement.
  ## Replicated Requests
    * You can replicate requests to multiple handlers and one of them may return faster than the other ones.
      * The downside is that you'll have to utilize resources to keep multiple copies of the handlers running.
    * An example of replicating a simulated request over 10 handlers:
      *
```Go
doWork := func(
  done <-chan interface{},
  id int,
  wg *sync.WaitGroup,
  result chan<- int,
) {
  started := time.Now()
  defer wg.Done()
  
  simulatedLoadTime := time.Duration(1+rand.Intn(5))*time.Second
  select {
  case <-done:
  case <-time.After(simulatedLoadTime):
  }
  
  select {
  case <-done:
  case result <- id:
  }
  
  took := time.Since(started)
  // Display how long handler would have taken
  if took < simulatedLoadTime {
  took = simulatedLoadTime
  }
  fmt.Printf("%v took %v\n", id, took)
}

done := make(chan interface{})
result := make(chan int)

var wg sync.WaitGroup
wg.Add(10)

for i := 0; i < 10; i++ {
  go doWork(done, i, &wg, result)
}

firstReturned := <- result
close(done)
wg.Wait()

fmt.Printf("Received an answer from #%v\n", firstReturned)```
    * All handlers must have equal opportunity to service the request.
    * If your handlers are too uniform, the chances that any one will finish quicker is smaller. You should only replicate requests to handlers that have different runtime conditions: difference processes, machines, paths to a data store, or access to different data stores altogether.
  ## rate limiting
    * rate limiting constrains the number of times some kind of resource is accessed to a finite number per unit time.
    * rate limiting allows you to reason about the performance and stability of your system by preventing it from falling outside boundaries you've already investigated.
    * Most often done with an algorithm call the token bucket.
      * The bucket has a starting depth (or burst) and a rate of replenishment.
    * Bursts are nice to have for users who access the system intermittently, but want to round-trip and quickly as possible when they do.
    * `golang.org/x/time/rate` implements a token bucket algorithm.
    * It's likely you'll want to establish tiers of rate limiting: per second, minute, day, etc.:
      *
```Go
type RateLimiter interface {
  Wait(context.Context) error
  Limit() rate.Limit
}

func MultiLimiter(limiters ...RateLimiter) *multiLimiter {
  byLimit := func(i, j int) bool {
  return limiters[i].Limit() < limiters[j].Limit()
  }
  sort.Slice(limiters, byLimit)
  return &multiLimiter{limiters: limiters}
}

type multiLimiter struct {
  limiters []RateLimiter
}

func (l *multiLimiter) Wait(ctx context.Context) error {
  for _, l := range l.limiters {
  if err := l.Wait(); err != nil {
    return err
  }
  }
  return nil
}

func (l *multiLimiter) Limit() rate.Limit {
  return l.limiters[0].Limit()
}```
  ## Healing Unhealthy Goroutines
    * In a long-running process, it's useful to create a mechanism that ensures goroutines are healthy and restarts them if not.
    * Uses the hearbeats pattern.
      * If your goroutine can become livelocked, make sure that the heartbeats contain information indicating that the goroutine is not only up, but doing useful work.
    * The logic that monitors a goroutine's health is called a steward, and the goroutine it monitors is called a ward.
    *
```Go
type startGoroutineFn func(
  done <-chan interface{},
  pulseInterval time.Duration,
) (heartbeat <-chan interface{})

newSteward := func(
  timeout time.Duration,
  startGoroutine startGoroutineFn,
) startGoroutineFn {
  return func(
  done <-chan interface{},
  pulseInterval time.Duration,
  ) (<-chan interface{}) {
  heartbeat := make(chan interface{})
  go func() {
    defer close(heartbeat)
    
    var wardDone chan interface{}
    var wardHeartbeat <-chan interface{}
    startWard := func() {
    wardDone = make(chan interface{})
    wardHeartbeat = startGoroutine(or(wardDone, done), timeout/2)
    }
    startWard()
  pulse := time.tick(pulseInterval)
  
  monitorLoop:
  for {
    timeoutSignal := time.After(timeout)
    
    for {
      select {
      case <- pulse:
      select {
      case heartbeat <- struct{}{}:
      default:
      }
      case <- wardHeartbeat:
      continue monitorLoop
      case <-timeoutSignal:
      log.Println("steward: ward unhealthy; restarting")
      close(wardDone)
      startWard()
      continue monitorLoop
      case <-done:
      return
      }
    }
    }
    
  }()
  return heartbeat
  }
}```
    *
```Go
doWork := func(done <-chan interface{}, _ time.Duration) <-chan interface{} {
  log.Println("ward: Hello, I'm irresponsible!")
  go func() {
  <-done
  log.Println("ward: I am halting.")
  }()
  return nil
}
doWorkWithSteward := newSteward(4*time.Second, doWork)

done := make(chan interface{})
time.AfterFunc(9*time.Second, func() {
  log.Println("main: halting steward and ward.")
  close(done)
})

// Listen until the steward's heartbeat stops
for range doWorkWithSteward(done, 4*time.Second) {}
log.Println("Done")```
    * To implement a ward that can return results:
      *
```Go
doWorkFn := func(
  done <-chan interface{},
  intList ...int,
) (startGoroutineFn, <-chan interface{}) {
  intChanStream := make(chan (<-chan interface{}))
  intStream := bridge(done, intChanStream)
  
  doWork := func(
  done <-chan interface{},
  pulseInterval time.Duration,
  ) <-chan interface {
  intStream := make(chan interface{})
  heartbeat := make(chan interface{})
  
  go func() {
    defer close(intStream)
    select {
    case intChanStream <- intStream:
    case <-done:
    return
    }
    
    pulse := time.Tick(pulseInterval)
    
    for {
    valueLoop:
    for _, intVal := range intList {
      if intVal < 0 {
      log.Printf("negative value: %v\n", intVal)
      return
      }
      
      for {
      select {
      case <-pulse:
        select {
        case heartbeat <- struct{}{}:
        default:
        }
      case intStream <- intVal:
        continue valueLoop
      case <-done:
        return
      }
      }
    }
    }
  }()
  return heartbeat
  }
  
  return doWork, intStream
}```
      *
```Go
done := make(chan interface{})
defer close(done)

doWork, intStream := doWorkFn(done, 1, 2, -1, 3, 4, 5)
doWorkWithSteward := newSteward(1*time.Millisecond, doWork)
doWorkWithSteward(done, 1*time.Hour)```
        * We're not doing anything with the heartbeat channel returned by `doWorkWithSteward`, so set `pulseInterval` to a large `time.Duration`.
      * When developing wards, be sure to account for systems that are sensitive to duplicate values when goroutines restart.
        *
```Go
  valueLoop:
  for {
    intVal := intList[0]
    intList := intList[1:]
    // ...
    }```
# Chapter 6: Goroutines and the Go Runtime
  ## work stealing
    * In a fork-join model, tasks are likely dependent on one another. Naively splitting them between processes will likely cause one of the processes to be underutilized, and lead to poor cache locality, as tasks that require the same data are scheduled on other processors.
    * Using a centralized FIFO queue where processors dequeue tasks as they have capacity or block on joins is better than simply dividing tasks among processors.
      * The queue is a shared critical section which is costly to access safely.
      * cache locality is worse: the queue must be loaded into the processor's cache every time it wants to queue or dequeue a task.
    * Solution is to give each processor its own deque of work and implement a work stealing algorithm.
    * A work stealing algorithm follows these rules:
      * At a fork point, add tasks to the tail of the deque associated with the thread.
      * If the thread is idle, steal work from the head of the deque associated with some other random thread.
      * At a join point that cannot be realized yet (i.e. the goroutine it is synchronized with has not completed yet), pop work off the tail of the thread's own deque.
      * If the thread's deque is empty, either:
        * Stall at a join.
        * Steal work from the head of a random thread's deque.
    * The work on the tail of a deque has interesting properties:
      * It is the work most likely needed to complete the parent's join.
      * It is the work most likely to be in our processor's cache.
    ### Stealing Tasks or Continuations?
      * goroutines are tasks.
      * Everything after a goroutine is a continuation.
      * Go's work stealing algorithm enqueues and steals continuations.
      * When a thread reaches an unrealized join point, it must pause execution and steal work. This is called a stalling join.
        * Stealing continuations causes stalls to occur significantly less often.
        * It is likely that your program will want to execute a function in a goroutine as soon as it is created.
        * It is also likely that the continuation from that goroutine will at some point want to join with that goroutine.
        * It's common for the continuation to attempt a join with the goroutine before it has finished.
        * It therefore makes sense to immediately begin working on the goroutine and enqueue the continuation.
      * By pushing the continuation onto the tail of the deque, it is least likely to get stolen by a thread that is popping from the head of the deque.
        * Becomes likely that the same thread will be able to pick the continuation up as soon as it finishes executing the goroutine.
      * **Stealing continuations**
        * Queue size: Bounded
        * Order of execution: Serial
        * Join point: Non-stalling
      * **Stealing tasks**
        * Queue size: Unbounded
        * Order of execution: Out of order
        * Join point: Stalling
      * Go's scheduler has 3 main concepts:
        * $$G$$: a goroutine.
        * $$M$$: an OS thread (called a "machine" in the source code).
        * $$P$$: a context (called a "processor" in the source code).
      * In Go's runtime, $$M$$s are started, which host $$P$$s, which then schedule and host $$G$$s.
      * The GOMAXPROCS setting controls how many contexts are available for use by the runtime.
        * Default setting is one context per logical CPU on the host machine.
      * There may be more or less OS threads than cores to help Go's runtime manage things like garbage collection and goroutines.
        * There will always be at least enough OS threads available to handle hosting every context.
      * The runtime maintains a thread pool for threads that aren't currently in use.
      * When a goroutine is blocked by input/output or syscalls, Go dissociates the context from the OS thread and hands it off to an unblocked thread, allowing it to continue scheduling goroutines.
        * The blocked goroutine remains associated with the blocked thread.
        * When it becomes unblocked, the host OS thread attempts to steal back a context from one of the other OS threads so it can continue executing the previously blocked goroutine.
          * If not possible, it will place its goroutine on a global context, and the thread will go to sleep and be place in the runtime's pool.
          * Contexts periodically check the global context.
          * If a context's queue is empty, it will try to steal from the global context first.
      * Go allows goroutines to be preempted during any function call.
        * An exception is goroutines that perform no input/output, syscalls, or function calls.
          * Can cause long garbage collector waits or deadlocks.
          * Very rare occurrence.
