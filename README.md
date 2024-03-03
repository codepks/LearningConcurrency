# LearningConcurrency

#Producer and Consumer Problem

Statement : We will take in some values , accumulate in a queue buffer and show it in the consumer buffer

```
#include <iostream>
#include <vector>
#include <queue>
#include <future>
#include <mutex>
#include <condition_variable>

std::condition_variable cv;
std::mutex mtx_;

//Producer object is responsible for producing values
class Producer{
public:

	void produceBufferValues(std::queue<int> &producedQueue)	{
		std::vector<int> vecVevalue{1, 2, 3, 4, 56, 7, 8, 9, 11, 12, 13, 14};
		for (size_t i = 0; i < vecVevalue.size(); i++)		{
			std::unique_lock<std::mutex> lock(mtx_);
			producedQueue.push(vecVevalue[i]);

			if (producedQueue.size() >= 10)
				cv.notify_one();
		}
	}
};


class Consumer{
public:
	void consumeBufferValues(std::queue<int> &consumedQueue)	{
		std::unique_lock<std::mutex> lock(mtx_);
		
		cv.wait(lock, [&consumedQueue]() {return !consumedQueue.empty(); });

		while (!consumedQueue.empty())		{
			std::cout << consumedQueue.front() << std::endl;
			consumedQueue.pop();
		}
	}
};

int main(){
	//Make a Producer object
	Producer producerObject;
	std::queue<int> dataQueue;

	//Run the producerObject's produceBufferValues function in a thread
	std::future<void> future_ = std::async(std::launch::async,
		&Producer::produceBufferValues, &producerObject, std::ref(dataQueue));

	Consumer consumerObject;
	std::this_thread::sleep_for(std::chrono::milliseconds(100));

	//Make a Consumer object
	consumerObject.consumeBufferValues(dataQueue);
}
```
