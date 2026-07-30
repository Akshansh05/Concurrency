# Java Concurrency Quick Reference

## 1. THE CORE PROBLEM
Multiple threads access the same variable → **race condition** → unpredictable results

```
Thread 1: read count (5)
Thread 2: read count (5)
Thread 1: count++ → 6
Thread 2: count++ → 6
Expected: 7, Got: 6 (Lost Update)
```

**Solution**: Only one thread can access the variable at a time (mutual exclusion)

---

## 2. LOCK MECHANISMS: WHEN TO USE WHAT

### synchronized (Old way)
```java
public synchronized void increment() {
    count++;
}
// OR
synchronized(this) {
    count++;
}
```

| Pros | Cons |
|------|------|
| Simple syntax | Can't timeout |
| JVM optimizes it | Can't interrupt |
| | No fairness control |

**Use when**: Simple critical sections, legacy code

---

### ReentrantLock (Modern way)
```java
private final ReentrantLock lock = new ReentrantLock();

public void increment() {
    lock.lock();
    try {
        count++;
    } finally {
        lock.unlock();  // ALWAYS in finally!
    }
}
```

| Pros | Cons |
|------|------|
| tryLock(timeout) | More verbose |
| lockInterruptibly() | Must unlock manually |
| Fair mode | More overhead than synchronized |
| Works with Condition | |

**Use when**: Need timeout, need interruption, need Condition (producer-consumer)

**Key methods:**
- `lock()` - acquire, block if not available
- `tryLock()` - non-blocking attempt
- `tryLock(timeout, unit)` - attempt with timeout
- `lockInterruptibly()` - can be interrupted while waiting
- `newCondition()` - create Condition for await/signal

---

### AtomicXXX (Lock-free)
```java
private final AtomicInteger count = new AtomicInteger(0);

count.incrementAndGet();        // No locks needed
count.compareAndSet(5, 6);      // CAS operation
```

| Pros | Cons |
|------|------|
| No blocking | Only single variables |
| Better under contention | Can't protect multiple vars together |
| Faster | CAS loop if collision |

**Use when**: Simple read-modify-write on ONE variable

---

## 3. CONCURRENT DATA STRUCTURES: DECISION TABLE

### Do I have multiple pieces of state that must change together?
```
YES → ReentrantLock (group multiple operations atomically)
NO  → Go to next question
```

### Is it a single variable?
```
YES + simple operation → AtomicInteger/Long/Reference
NO  → Go to next question
```

### Is it a Map?
```
YES → ConcurrentHashMap
NO  → Go to next question
```

### Is it a List/Collection?
```
Many readers, few writers → CopyOnWriteArrayList
Producer-consumer pattern → BlockingQueue
Unbounded, lock-free → ConcurrentLinkedQueue
Sorted → ConcurrentSkipListMap/Set
```

---

## 4. CONCURRENT DATA STRUCTURES: DETAILED COMPARISON

| Structure | Use Case | Locking | Ordering |
|-----------|----------|---------|----------|
| **ConcurrentHashMap** | Multi-threaded map, different keys | Per-bucket locks | None |
| **BlockingQueue** | Producer-consumer | Internal locks | FIFO (usually) |
| **PriorityBlockingQueue** | Priority tasks | Internal lock | By priority |
| **DelayQueue** | Delayed tasks | Internal lock | By delay time |
| **CopyOnWriteArrayList** | Heavy reads, few writes | Write locks all | Snapshot iteration |
| **ConcurrentLinkedQueue** | Unbounded queue | Lock-free (CAS) | FIFO |
| **ConcurrentSkipListMap** | Sorted concurrent map | Lock-free (CAS) | Sorted |
| **ConcurrentLinkedDeque** | Double-ended queue | Lock-free | Both ends |

---

## 5. THREADPOOLEXECUTOR: HOW IT WORKS

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    2,                                    // corePoolSize
    5,                                    // maxPoolSize
    60,                                   // keepAliveTime
    TimeUnit.SECONDS,                     // time unit
    new LinkedBlockingQueue<>(10),        // task queue
    Executors.defaultThreadFactory(),     // thread factory
    new ThreadPoolExecutor.AbortPolicy()  // rejection policy
);
```

### Task Submission Flow:
```
Task arrives
  ↓
Active threads < corePoolSize?
  YES → Create core thread, run immediately
  NO  → Queue has space?
        YES → Add to queue
        NO  → Active threads < maxPoolSize?
              YES → Create non-core thread, run immediately
              NO  → Apply rejection policy
```

### corePoolSize vs maxPoolSize
- **corePoolSize**: Threads kept alive even when idle
- **maxPoolSize**: Maximum threads ever created
- **Queue**: Buffer between core and max threads

### Rejection Policies:
- **AbortPolicy** (default): Throw RejectedExecutionException
- **DiscardPolicy**: Silently drop task (lose work!)
- **DiscardOldestPolicy**: Remove oldest task, add new one
- **CallerRunsPolicy**: Execute in calling thread (blocks caller)

### Shutdown:
```java
// Graceful: stop accepting new, wait for queued to finish
executor.shutdown();
executor.awaitTermination(10, TimeUnit.SECONDS);

// Immediate: interrupt all, return unstarted
List<Runnable> unstarted = executor.shutdownNow();
```

---

## 6. SYNCHRONIZATION PRIMITIVES: COORDINATION TOOLS

### CountDownLatch (one-time use)
```java
CountDownLatch latch = new CountDownLatch(3);
// 3 threads call latch.countDown()
// Main thread calls latch.await() and waits until counter = 0
```
**Use**: Wait for N tasks to complete

---

### CyclicBarrier (reusable)
```java
CyclicBarrier barrier = new CyclicBarrier(3);
// 3 threads call barrier.await()
// All wait until 3 threads reach it, then all proceed together
```
**Use**: N threads coordinate at checkpoint, repeatable

---

### Semaphore (resource limiting)
```java
Semaphore semaphore = new Semaphore(2);
// semaphore.acquire() - wait if no permits
// semaphore.release() - release a permit
```
**Use**: Limit access to N resources (thread pool bouncer)

---

### ReadWriteLock (read-heavy workloads)
```java
ReadWriteLock rwLock = new ReentrantReadWriteLock();
// Multiple readers can acquire read lock simultaneously
// Only one writer, blocks all readers
rwLock.readLock().lock();   // Many threads can hold this
rwLock.writeLock().lock();  // Only one thread can hold this
```
**Use**: Cache with many readers, few writers

---

## 7. RESOURCE SHARING: HOW TO DO IT RIGHT

### Pattern 1: Producer-Consumer (BlockingQueue)
```java
BlockingQueue<Integer> queue = new LinkedBlockingQueue<>(10);

// Producer
queue.put(value);           // Blocks if queue full

// Consumer
Integer value = queue.take(); // Blocks if queue empty
```

### Pattern 2: Reader-Writer Cache (ConcurrentHashMap + ReadWriteLock)
```java
ConcurrentHashMap<String, Value> cache = new ConcurrentHashMap<>();

// For grouped operations (multiple map operations together):
ReadWriteLock lock = new ReentrantReadWriteLock();
lock.readLock().lock();     // Many readers
// vs
lock.writeLock().lock();    // One writer
```

### Pattern 3: Task Queue with Fixed Workers
```java
ExecutorService executor = Executors.newFixedThreadPool(3);
BlockingQueue<Task> taskQueue = new LinkedBlockingQueue<>(100);

// Worker loop:
while (true) {
    Task task = taskQueue.take();  // Blocks until task available
    task.run();
}
```

### Pattern 4: Bounded Resource Pool (Semaphore)
```java
Semaphore permits = new Semaphore(10);  // 10 resources
permits.acquire();   // Wait if none available
try {
    useResource();
} finally {
    permits.release();
}
```

---

## 8. COMMON MISTAKES (GOTCHAS)

### ❌ Mistake 1: Forgetting finally block
```java
lock.lock();
// Do work
lock.unlock();  // If work throws exception, never reaches here!
```

### ✅ Fix:
```java
lock.lock();
try {
    // Do work
} finally {
    lock.unlock();  // Always runs
}
```

---

### ❌ Mistake 2: Swallowing InterruptedException
```java
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    // Silently ignoring - others don't know thread was interrupted
}
```

### ✅ Fix:
```java
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();  // Re-set flag
    // OR throw new RuntimeException(e);
}
```

---

### ❌ Mistake 3: Unbounded queues (memory leak)
```java
BlockingQueue<Task> queue = new LinkedBlockingQueue<>();  // Unbounded!
```

### ✅ Fix:
```java
BlockingQueue<Task> queue = new LinkedBlockingQueue<>(1000);  // Bounded
```

---

### ❌ Mistake 4: Non-atomic grouped operations
```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// Not atomic:
if (!map.containsKey("key")) {
    map.put("key", value);  // Another thread might insert between check and put
}
```

### ✅ Fix:
```java
// Atomic:
map.putIfAbsent("key", value);

// Or for more complex logic:
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    if (!map.containsKey("key")) {
        map.put("key", value);
    }
} finally {
    lock.unlock();
}
```

---

## 9. DECISION FLOWCHART

```
Need to protect shared mutable state?

Multiple pieces that must change together?
  YES → ReentrantLock + try/finally
  
  NO → Single variable, simple operation?
    YES → AtomicInteger/Long/Reference
    
    NO → Is it a Map?
      YES → ConcurrentHashMap
      
      NO → Is it a List/Queue/Set?
        YES, Producer-Consumer → BlockingQueue
        YES, Heavy reads → CopyOnWriteArrayList
        YES, Sorted → ConcurrentSkipListMap
        YES, Lock-free → ConcurrentLinkedQueue
        
        NO → Use synchronized (for simple objects)
```

---

## 10. WHEN EACH PRIMITIVE BLOCKS/WAKES UP

| Primitive | Blocks on | Wakes on |
|-----------|-----------|----------|
| `lock()` | Acquire | Lock released |
| `tryLock(timeout)` | Acquire | Lock released OR timeout |
| `lockInterruptibly()` | Acquire | Lock released OR interrupted |
| `queue.take()` | Empty queue | Element added |
| `queue.put()` | Full queue | Element removed |
| `latch.await()` | Counter > 0 | countDown() reaches 0 |
| `barrier.await()` | Waiting for others | All threads arrive |
| `semaphore.acquire()` | No permits | release() called |
| `condition.await()` | Wait | signal() or signalAll() |

---

## 11. THREAD LIFECYCLE IN EXECUTOR

```
ExecutorService executor = Executors.newFixedThreadPool(2);

// State 1: RUNNING (2 threads alive, waiting for tasks)
executor.submit(() -> System.out.println("Task 1"));
executor.submit(() -> System.out.println("Task 2"));

// State 2: SHUTDOWN (stop accepting new tasks)
executor.shutdown();
// Existing tasks + queued tasks run, but no new submissions

// State 3: TERMINATED (all tasks done, all threads dead)
executor.awaitTermination(10, TimeUnit.SECONDS);  // Wait for this

// During shutdown, can't call executor.submit() — throws RejectedExecutionException
```

---

## 12. QUICK LOOKUP: "Which structure for this use case?"

| Use Case | Structure | Why |
|----------|-----------|-----|
| Share tasks between threads | BlockingQueue | Blocks if empty/full, coordinates naturally |
| Cache data | ConcurrentHashMap | No locks for different keys |
| Limit access to N resources | Semaphore | Exactly designed for this |
| Wait for multiple tasks | CountDownLatch | Simple one-time synchronization |
| Run task after delay | DelayQueue | Orders by delay time automatically |
| Heavy reads, few writes | ReadWriteLock + Cache | Readers don't block each other |
| Publish-Subscribe | ConcurrentHashMap + BlockingQueue | Topics → subscriber queues |
| Counter multiple threads increment | AtomicInteger | No locks, CAS operations |
| Guarantee ordered processing | ExecutorService + single thread | Single worker processes queue sequentially |
| Priority queue processing | PriorityBlockingQueue | Orders by priority automatically |

---

## Practice Template

For each interview problem, ask yourself:

```
1. What is the shared mutable state?
   - List all variables that multiple threads access

2. What synchronization primitive protects it?
   - Atomic? Lock? Concurrent collection?

3. What's the thread lifecycle?
   - How many threads?
   - When do they start/stop?
   - Who creates them?

4. What's the coordination mechanism?
   - BlockingQueue? Semaphore? CountDownLatch?

5. What happens on failure/shutdown?
   - Exception handling?
   - Resource cleanup?
   - No leaked threads?
```

Apply this to every LLD concurrency problem you solve.
