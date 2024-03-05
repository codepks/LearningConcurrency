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


## Concurrency and Parallelism <br>
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

## Creating Threads

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

## Waiting, Killing, and Detaching

### Why join() <br>
- join() blocks the the current thread until worker threa's job is completed.
- In C++ one must specify what happens to a thread when it goes out of scope.
- It is safe to either detach them or wait for their completion by *joining* them.

<br>
One must make sure that **its destructor is not called when it is still joinable**( joinable means it is not detached or killed).

<br> 
If you have not detached or joined then it will call std::terminate <br>
Most commonly, you should use join to ensure proper thread termination and resource cleanup, especially for critical tasks. <br>
source : https://stackoverflow.com/questions/27392743/c11-what-happens-if-you-dont-call-join-for-stdthread


### Kill a thread

```std::terminate()``` can kill entire program process, better use ```return```

### Detach a thread
**What Happens** <br>
- When you detach a thread the program no longer manages its lifecycle.
- Useful when threre is some background task supposed to run and requires no interaction with main thread

**Usages**
- It is often used to avoid deadlocks when main function needs to continue doing its operation without waiting for a thread operation.

**Drawbacks**
- Once a thread is detached its resource cleanup cannot be guaranteed leading to unexpected behaviour.
- It is difficult to debug them as there is no return value from a detached thread.
