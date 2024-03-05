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











