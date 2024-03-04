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
```

Working of cv.wait()
 cv.notify keeps the lock in released state or basically doesn't acquire lock until the condition is satified.
 Till then it is in sleep mode.
