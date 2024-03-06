# LearningConcurrency

# Producer and Consumer Problem

Statement : We will take in some values , accumulate in a queue buffer and show it in the consumer buffer

```
#include <iostream>
#include <queue>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <future>

std::queue<int> buffer;
std::mutex mtx;
std::condition_variable cv;

class Producer {
public:
    void producer() {
        for (int i = 1; i <= 5; ++i) {
            std::lock_guard<std::mutex> lock(mtx);
            buffer.push(i);
            std::cout << "Produced: " << i << std::endl;
            cv.notify_one();
            std::this_thread::sleep_for(std::chrono::milliseconds(500));
        }
    }
};

class Consumer {
public:
    void consumer() {
        while (true) {
            std::unique_lock<std::mutex> lock(mtx);
            cv.wait(lock, [] { return !buffer.empty(); });
            int data = buffer.front();
            buffer.pop();
            std::cout << "Consumed: " << data << std::endl;
            lock.unlock();
            std::this_thread::sleep_for(std::chrono::milliseconds(1000));
        }
    }
};

int main() {
    Producer producer;
    Consumer consumer;
    std::future<void> future1 = std::async(std::launch::async, &Producer::producer, &producer);
    std::future<void> future2 = std::async(std::launch::async, &Consumer::consumer, &consumer);

    return 0;
}

OUTPUT :
Produced: 1
Produced: 2
Produced: 3
Produced: 4
Produced: 5
Consumed: 1
Consumed: 2
Consumed: 3
Consumed: 4
Consumed: 5

```

*Working of cv.wait()*
    cv.wait keeps the lock in released state or basically doesn't acquire lock until the condition is satified. 
    Till then it is in sleep mode.
*Working of  cv.notify()*
    cv.notify wakes up cv.wait, but it is a good practice to unlock the thread before we let the wait thread acquire the lock.

### More Synchronous

```
class Producer {
public:
    void producer() {
        for (int i = 1; i <= 5; ++i) {
            std::unique_lock<std::mutex> lock(mtx);
            buffer.push(i);
            std::cout << "Produced: " << i << std::endl;

            //we must unlock so that cv.nofity actually wakes up the the thread wait
            lock.unlock();        

            cv.notify_one();

            //make sure we are waiting till the producer produces another data
            std::this_thread::sleep_for(std::chrono::milliseconds(500));
        }
    }
};

class Consumer {
public:
    void consumer() {
        while (true) {
            std::unique_lock<std::mutex> lock(mtx);
            cv.wait(lock, [] { return !buffer.empty(); });
            int data = buffer.front();
            buffer.pop();
            std::cout << "Consumed: " << data << std::endl;
            lock.unlock();

            //removed sleep so that the data produced by the Producer is quickly consumed
        }
    }
};


OUTPUT:
Produced: 1
Consumed: 1
Produced: 2
Consumed: 2
Produced: 3
Consumed: 3
Produced: 4
Consumed: 4
Produced: 5
Consumed: 5

```


# Concurrency and Parallelism <br>
source : https://github.com/methylDragon/coding-notes/blob/master/C++/07%20C++%20-%20Threading%20and%20Concurrency.md

**Concurrency** <br>
- When tasks are run simultaneously but not necessarily at the same physical time. <br>
- It is achieved through multithreading or asynchronous programming.<br>
- It can be done on a single core or multicore processor.<br>
- *Example* : It can be achieved through threads, async, mutexes<br>

**Parallelism**<br>
- Running tasks simulataneously on multiple cores or multiple processors.<br>
- Parallelism requires hardware support in form of multiple cores or multiple processors.<br>
- Here we are dividing tasks into subtasks and execute concurrently on multiple processors.<br>
- We exploit the computation power of the processors fully here.<br>
- *Example* : Using OpenMP parallel programming:<br>
- https://www.geeksforgeeks.org/introduction-to-parallel-programming-with-openmp-in-cpp/


**Maximum number of threads for a given hardware**
unsigned int c = std::thread::hardware_concurrency();

Creating more threads than these doesn't benefit anyone.

# Creating Threads

There are several ways to create a thread:
- Using a function pointer
- Using a lambda function
- Using a functor

## this_thread

Refers to the current thread:

```
std::this_thread::get_id();
std::this_thread::yield();
std::this_thread::sleep_for(std::chrono::seconds(1));
std::this_thread::sleep_until(time_point);
```

## Pass by reference

You need to explicitly wrap the arguments in std::ref() to pass by reference.

```
void ref_function(int &a, int b) {}
int val;
std::thread ref_function_thread(ref_function, std::ref(val), 2);
```
*Because the thread functions can't return anything, passing by reference is the only way to properly get data out of a thread without using global variables*

## thread_local

source : https://www.geeksforgeeks.org/thread_local-storage-in-cpp-11/
thread_local are like static variables for threads and exists till the threads exists.

better usage: 
```
// Suppose this is your thread function
void method()
{
  static int var = 0;
  var++;
}
```
In the code above var will increment with the same instance across all the threads.
But if you want to have your own copy of static_variable that is local to a thread then one should use the function below:

```
void method()
{
  thread_local int var = 0;
  var++;
}
```

# Waiting, Killing, and Detaching

## Why join() <br>
- join() blocks the the current thread until worker threa's job is completed.
- In C++ one must specify what happens to a thread when it goes out of scope.
- It is safe to either detach them or wait for their completion by *joining* them.

<br>
One must make sure that its destructor is not called when it is still joinable ( joinable means it is not detached or killed).

<br> 
If you have not detached or joined then it will call std::terminate <br>
Most commonly, you should use join to ensure proper thread termination and resource cleanup, especially for critical tasks. <br>
source : https://stackoverflow.com/questions/27392743/c11-what-happens-if-you-dont-call-join-for-stdthread


## Kill a thread

```std::terminate()``` can kill entire program process, better use ```return```

## Detach a thread
**What Happens** <br>
- When you detach a thread the program no longer manages its lifecycle.
- Useful when threre is some background task supposed to run and requires no interaction with main thread

**Usages**
- It is often used to avoid deadlocks when main function needs to continue doing its operation without waiting for a thread operation.

**Drawbacks**
- Once a thread is detached its resource cleanup cannot be guaranteed leading to unexpected behaviour.
- It is difficult to debug them as there is no return value from a detached thread.

```
void turnOnSimulator() {
        std::thread simulatorThread(&TemperatureSensorSimulator::generateTemperatures, this);
        simulatorThread.detach();  // Detach the thread to run independently
    }
```
- Detaching the thread allows it to run independently even after the main function finishes execution. This means the generateTemperatures method will continue generating temperatures in the background, and the main thread can proceed without waiting for the thread to finish.

- If we don't use detach the main thread will will call ```.join()``` implicitly at the end of ```turnOnSimulator()``` which will main thread to wait for thread to finish

## Data Race

Reading is always thread safe compared but writing is not.
Result is inconsistent. Put lock to get consistent results.

## Atomic Operation
- It guarantees that no race condition will occur
- Should only be used when we need them
  
### Sample Code
```
std::atomic_int acnt;
int cnt;
 
void f(){
    for (int n = 0; n < 10000; ++n)    {
        ++acnt;
        ++cnt;        
    }
}
 
int main(){  
    std::vector<std::jthread> pool;
    for (int n = 0; n < 10; ++n)
        pool.emplace_back(f);    
 
    std::cout << "The atomic counter is " << acnt << '\n'
              << "The non-atomic counter is " << cnt << '\n';
}


Possible output:

The atomic counter is 100000
The non-atomic counter is 69696
```

## Atomic Types

**Types**
```
// source : https://en.cppreference.com/w/cpp/atomic/atomic

atomic_bool        std::atomic<bool>
atomic_char        std::atomic<char>
```

**Operational functions**

```
std::atomic class, such as load(), store(), exchange(), compare_exchange_weak(), compare_exchange_strong(), fetch_add(), fetch_sub(), etc
```

**Memory ordering**

- Atomic variables support different memory orderings, which specify the ordering constraints for memory operations involving the atomic variable.
- The memory orderings include memory_order_relaxed, memory_order_acquire, memory_order_release, memory_order_acq_rel, and memory_order_seq_cst.


# Mutex

- They are there to prevent race conditions
- Overuse of locks can lead to deadlock situations

## Avoiding Deadlock
Let deadlock occur, then do preemption to handle it once occurred.

## Using locks

This code below is not a correct way but using using unique_lock and lock_guards is always better as they follow RAII.
```
std::mutex my_mutex;

thread_function(){
  my_mutex.lock(); // Acquire lock
  // Do some non-thread safe stuff...
  my_mutex.unlock(); // Release lock
}
```

# Lock_Guard Types

## lock_guard
Releases lock once it goes out of scope.
```
std::mutex my_mutex;
 
thread_function()
{
  std::lock_guard<std::mutex> guard(my_mutex); // Acquire lock
  // Do some non-thread safe stuff...
}
```

## scoped_lock
From C++ 17 <br>
It can take multiple mutexes

```
std::scoped_lock<std::mutex, std::mutex> guard(mutex_1, mutex_2);
```

## unique_lock

By default behaves as lock_guard but comes with various functionalities

```
std::unique_lock<std::mutex> guard(my_mutex);

// Check if guard owns lock (either works)
guard.owns_lock();
bool(guard);

// Return function without releasing the lock
return std::move(guard);

// Release lock before destruction
guard.unlock();
```

**defering**

```
// Initialise the lock guard, but don't actually lock yet
std::unique_lock<std::mutex> guard(mutex_1, std::defer_lock);

// Now you can do some of the following!
guard.lock(); // Lock now!
guard.try_lock(); // Won't block if it can't acquire
guard.try_lock_for(); // Only for timed_mutexes
guard.try_lock_until(); // Only for timed_mutexes
```

## share_lock 

Just like unique lock except that it works for **shared_mutex**

```
std::shared_lock my_mutex;
std::shared_lock<std::shared_mutex> guard(my_mutex);

// Check if guard owns lock (either works)
guard.owns_lock();
bool(guard);

// Return function without releasing the lock
return std::move(guard);

// Release lock before destruction
guard.unlock();
```

```
// Initialise the lock guard, but don't actually lock yet
std::shared_lock<std::shared_mutex> guard(mutex_1, std::defer_lock);

// Now you can do some of the following!
guard.lock(); // Lock now!
guard.try_lock(); // Won't block if it can't acquire
guard.try_lock_for(); // Only for timed_mutexes
guard.try_lock_until(); // Only for timed_mutexes
```

# Lock Types

## Exclusive lock

```
std::mutex
```
Exclusive locks blocks both read and write. Only one thread can access a resource at a time.

## Shared Lock

```
std::shared_mutex
```

Multiple locks can acquire access but only for read access. In order to write over the resource one needs to have exclusive lock instead.


```
std::shared_mutex mtx; // Shared mutex object

// Function to simulate reading data
void readData() {
  mtx.lock_shared(); // Acquire a shared lock
  std::cout << "Reading data..." << std::endl;
  std::this_thread::sleep_for(std::chrono::milliseconds(500)); // Simulate reading time
  mtx.unlock_shared(); // Release the shared lock
}

// Function to simulate writing data
void writeData() {
  mtx.lock(); // Acquire an exclusive lock
  std::cout << "Writing data..." << std::endl;
  std::this_thread::sleep_for(std::chrono::milliseconds(1000)); // Simulate writing time
  mtx.unlock(); // Release the exclusive lock
}

int main() {
  std::thread t1(readData);
  std::thread t2(readData);
  std::thread t3(writeData);

  t1.join();
  t2.join();
  t3.join();

  return 0;
}
```

# Mutex Types

**std::mutex**
Plain one

**std::timed_mutex**
Used with ```try_lock_for()``` and ```try_lock_until()```

**std::recursive_timed_mutex**

**std::recursive_timed_mutex**

**std::shared_timed_mutex**

# Condition Variables
For signal even handling

**Example Flow**
> - Thread acquires lock
> - Check if condition is false
> - If false, call wait(), which releases the lock and blocks the thread until the condition is fulfilled
> - If a condition is fulfilled, the condition variable must be notified before it can check
> - Once the condition check succeeds, thread reacquires lock and continues execution

- Condition variable uses unique_lock. It can signal when lock is released
-  I think it's a good rule of thumb to avoid holding the lock associated with a condition variable while calling notify_one() or notify_all().
-  Keep in mind that the lock() call in the while loop is necessary at some point, because the lock needs to be held during the while (!done) loop condition check. But it doesn't need to be held for the call to notify_one().

> source
> https://stackoverflow.com/questions/17101922/do-i-have-to-acquire-lock-before-calling-condition-variable-notify-one

note: mutex unlock takes time

**Sample Code** <br>
Check producer and Consumer code in the top

**Avoid Spurious wake ups**<br>
Use lamdas
```
//If we want to just check a bool called condition we need to use lambdas
condition_var.wait(guard, [](){return condition == true;});
```

# Task based parallelism

- Task based parallelism is handled via ```std::async``` <br> 
- The main benefit of this form of parallelism is the great ease in getting returned values from tasks that you start
- Before, when using ```std::thread``` ,one needs to pass variables by reference

## Threads vs Tasks

Use threads if:
- You need tight control over mutexes
- Need to run long-lived, complex tasks

Use tasks if:
- You want fairly simple code and don't care for managing threads
- Are running short tasks

# Promises and Futures

They are introduced more likely because of the issue of return value with ```std::thread```

## Futures
- future variables are class template that will asssigned value in future and values to it are acessed via .get() . 
- .get() function blocks the current thread until the value is retrieved
- values to futures are assigned by asynchronous operations like ```std::async```, ```std::packaged_task``` and ```std::promise```

## Shared Futures

A std::shared_future works the same way, except it is copyable. Which means that multiple threads are allowed to wait for the same shared state. <br>


## Promises

- It is a writable container
- ```std::promise``` stores a value in an asynchronous operation which is later acquired by ```std::future```
- Every ```promise``` is associated with ```future```
- so unless ```.set()``` has been done on a promise, your ```.get()``` will be blocked on future

## Future

- A future acts like a read-only handle to the eventual result of a promise.
- Future and promises work on *pair* basis and are associate to each other

**Shared Future**

 Each future's ```get()``` method can only be called once. If you want a future that can be accessed multiple times, use a shared_future instead

 ## Work flow

> - A promise is created.
>  - An asynchronous operation (e.g., network request, file I/O) is initiated.
>  - The future is associated with the promise.
>  - When the operation finishes:
>  - If successful, the promise is resolved with the result, and the future can now be used to access the value.
>  - If unsuccessful, the promise is rejected with an error, and the future might indicate the error state.
>  - Code that needs the result can wait on the future or check its status to retrieve the final value.

**CODE SAMPLE 1**
```
// Source: https://thispointer.com//c11-multithreading-part-8-stdfuture-stdpromise-and-returning-values-from-thread/

void initiazer(std::promise<int> * promObj)
{
    std::cout<<"Inside Thread"<<std::endl;

    // 4. set the value to the promise
    promObj->set_value(35);
}
 
int main()
{
    //1. create a promise
    std::promise<int> promiseObj;

    //2. associate a future to the proimse
    std::future<int> futureObj = promiseObj.get_future();

    //3. send the promise to a separate thread
    std::thread th(initiazer, &promiseObj);

    //4. obtain the future 
    std::cout<<futureObj.get()<<std::endl;

    th.join();
    return 0;
}
```

**CODE SAMPLE 2 - without promise**

```
void produceValue(std::promise<int>& promise) {
   // Simulate some work
   std::this_thread::sleep_for(std::chrono::milliseconds(500));

   // Set the value of the promise
   promise.set_value(10);
}

int main() {
   std::promise<int> promise;
   // Use a regular future for this example
   // (You can explore shared futures later)
   std::future<int> future = promise.get_future();

   std::thread workerThread(produceValue, std::ref(promise));

   // Main thread waits for the value from the future
   int val = future.get(); // Blocks until the promise is set

   std::cout << "Value received: " << val << std::endl;

   workerThread.join(); // Ensure the worker thread finishes

   return 0;
}

```

