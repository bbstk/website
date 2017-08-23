---
title: "A Server Benchmark Story: Design, Implementation, and Results"
date: 2017-08-23T21:15:45+01:00
draft: false
---

*This blog post discusses the requirements of modern day server applications, what benchmarks can be used, and describes a specific example that is used to compare __Go__, __Erlang__, and __Scala with Akka__ for building such applications.*

## My server is better than yours

What does it mean for a server to be good? How is performance measured in such systems? In order to answer these questions, one needs to first understand what servers do. In its essence a server does three things - receives requests from clients, does some computation, and returns a response to the sender of the request. The computation depends on the specific application, but the rest of the steps are pretty much the same. 

Considering its purpose (serving requests from clients), it is easy to see what makes for a performant server. First, it needs to serve requests _**fast**_. The time between the client sending a request and receiving back a response has to be minimized. Second, it needs to be able to serve _**many**_ requests. An application serving two times more requests than another for the same period of time is considered more performant. These two performance measurements are known as *latency* and *throughput*, respectively.

The architecture of a server application is build to be able to deal with large number of requests in a timely manner. Figure 1 shows a simplified view of a client-server system. A server application consists of multiple processes. A dispatcher process receives the requests and gives it to a worker to process. Having multiple workers running at the same time allows for increasing the throughput of the server. Moreover, a highly concurrent application will utilize multi-core hardware better. This means that less machines are needed to support users, ultimately leading to reduced operation cost.

<figure>
	<img src="/images/server-architecture.jpg" alt="Simplified server architecture" style="width:50x; margin-left: auto; margin-right: auto; display: block;"/>
	<figcaption style="text-align: center;">Figure 1. Simplified server architecture</figcaption>
</figure>

The programming language used to build a server application needs to support the features required for this architecture. The ability to quickly create processes, to be able to maintain them, and to have quick communication between them plays a role in the performance of a server application. 

Another important feature for server application is the ability to handle failure. What is the effect of a failure in a Worker from Figure 1? How will performance suffer? What will happen if the Dispatcher fails? Using a programming language designed with fault tolerance in mind will make answering these questions easier and with less uncertainty.

## Picking the right tools for the job

Say you want to build an application serving millions of users. They will send you photos, you will process them using some fancy machine learning algorithm and will return to the users the number of cats in the photos. So how do you use choose a programming language for your application between C, Java, Go, Erlang, or any other? You use a benchmark! More concretely, you write the same benchmark in the languages that you want to compare, and then measure the performance of the different implementations. But what should the benchmark be? 

**The ideal benchmark** - Easy, write the whole, real application in C, Java, Go, Erlang, etc. Then compare the performance and you are done. You know the best language for your use case!  Of course, using this “benchmark” is almost never possible, or worthwhile the time and effort.

**The realistic benchmark** - Pick an aspect of your application that you think will be important for the overall performance. In this case, the machine learning algorithm that detects cats in photos seems like a good place to start. Implementing it in different languages and comparing the performance can give you an idea of what “tool” to pick.

However, if you are building a safety critical system like a railway signaling system, for example, benchmark the fault tolerance of the considered programming languages may be more important.

On the other hand, if you don’t have a specific application in mind, but want to compare languages on their general performance for building servers - you can benchmark what is common for most server applications - the concurrency patterns showed in Figure 1 and discussed above. This is what I did - a generic server benchmark implemented in Erlang, Go, and Scala with Akka. The design, implementation, and results from this benchmark are described in the next sections.

## Designing the benchmark

In order to compare languages for engineering server applications, I focused on the concurrency patterns described above. More specifically, I decided to test the process communication throughput of the languages. The benchmark needed to measure how many messages each language can send between pairs of processes per unit of time. Moreover, it needed to examine how the system scales with increase of the processes’ pairs and computational resources (processor cores).

The final design of my benchmark is shown in figure 2. It consists of 3 main entities:

*  A pool of workers, that are constantly communicating between each other.
*  An aggregator that periodically receives a report of the message count from each group and aggregates it.
*  A main process that is responsible for initializing, starting, and stopping the worker pool and the aggregator.
<figure>
	<img src="/images/benchmark-design.jpg" alt="Benchmark Design" style="width: 100%;"/>
	<figcaption style="text-align: center;">Figure 2. - The benchmark's design</figcaption>
</figure>

This design was born after a series of iterations, trying to replicate as closely as possible some of the concurrency requirements that modern day server applications have. Namely, fast message passing between processes.

In order to understand as much as possible about the languages that will implement it, the number of workers in the worker pool can vary. This way, one can see how the implementation scales with the addition of more processes. Another variable that can be set is the size of the message that the workers exchange. I wanted it to be realistic for a message exchanged in a server application, and picked the value of 500 bytes. However, one can try and experiment with different message sizes using this benchmark.

This design also allows for interesting observations when the benchmark is run on different number of processor cores. More about this can be seen in the section about running the benchmark and the one about results

## Getting your hands dirty - the implementation

The benchmark was implemented in Erlang, Go, and Scala with Akka. This section gives a brief overview of the implementation in each of the languages. The full implementation can be seen in this [GitHub repo](https://github.com/bbstk/server-languages-benchmarks/tree/master/ConcurrentProcessesThroughput).

*__Note:__* I am not an expert in these languages. More efficient implementations are more than likely to exist.

The implementation of the benchmark can be broken down into four stages.

* **Worker pair** 

The worker pair consists of two processes constantly exchanging a message between each other. After a certain number of messages have been exchanged, one of the processes notifies the aggregator, and returns to its never ending game of “hot potato” with its partner process.

Both Erlang and Scala with Akka are based on the [actor model](link: https://en.wikipedia.org/wiki/Actor_model). There the “actor” is the fundamental primitive of concurrent computation. Not surprise here, the worker processes here are regular actors. The way that messages are passed between actors is really straightforward - each actor has a “mailbox” and if you have a reference to that “mailbox” you can directly send messages to that actor.

Go’s concurrency is based on Tony Hoare’s [Communicating Sequential Processes](https://en.wikipedia.org/wiki/Communicating_sequential_processes). The main building blocks in the language’s concurrency are goroutines, which resemble lightweight threads, and channels, which are used for communication between the goroutines. Goroutines are used to represent the two workers in the Go’s implementation. The message that is constantly being passed between the two goroutines goes through a channel.

* **Creating the worker pool**

The number of processes to be created in the thread pool is specified on running the benchmark. In Erlang and Scala, when the processes are created, the process IDs and actor references, respectively, are stored in a list. In Go, however, there is no goroutine reference to be kept. Instead, all the worker processes that are spawned share a starter channel of type byte, to which the main process keeps a reference.

* **Starting the worker pool**

In the case of Erlang and Scala with Akka, the list of process ids or actor references is iterated and all all the actors are sent a start message. In Go the main process sends a byte over the starter channel for each process there is in the thread pool. 

* **Aggregator**

The aggregator in Erlang and Scala is a process or actor which responds to two messages. First, it sums up all the numbers that are sent to it. The second message it can receive is ”show” where the aggregator returns the current sum. In Go, the aggregator is a goroutine which always listens to a channel. It increments a variable with the values it receives from that channel. When the benchmark finishes, the main process prints the value of this variable.
It is important to note that timestamps are taken on start of the worker-client pairs and when the aggregator is asked for the sum. This way the integrity of the experiment can be tested by seeing exactly how much time of the worker-client communication has been accounted for.

## Running the benchmark

After the implementation is done, the time comes to actually see how the different languages compare. It is great to see the results of your work, and running the benchmark was definitely something I was looking for.

However, I wanted to do it right. I couldn’t just run it on my laptop once, and make a conclusion about what the best language is. I ran the benchmark on a node part of the [Glasgow Parallelism Group](http://www.dcs.gla.ac.uk/research/gpg/) cluster with 16 Intel cores (2 Intel Xeon E5-2640 2GHz), 64Gb RAM, 4Gb RAM/core, and running Scientific Linux 6.

Each implementation was tested for 60 seconds with different number of cores (1, 2, 4, 8, 16, and 32) and different number of paired processes (1, 2, 4, 6, 8, 16, 32, 64, and 128). The number of messages sent was recorded after each run. Moreover, I run every combination at least 3 times and took the median values in order to reduce the effect of outliers. 

I wrote a simple bash script to run all these different combinations for me, because otherwise it would have taken ages. I wanted to make sure that the different implementations are running on the same cores, and this is why I used the [taskset command](https://linux.die.net/man/1/taskset). Since Erlang and Scala are running on virtual machines, I also made sure to restart them before each run.

## The Results

Below are some graphs showing the findings from running the benchmark. The raw data, and some more graphs can be found together with the implementation in the [github repository](https://github.com/bbstk/server-languages-benchmarks/tree/master/ConcurrentProcessesThroughput).

Overall, this benchmark shows Go performing better than Erlang and Scala with Akka. This is shown across different number of cores and different number of paired processes. On one core Go shows almost twice better throughput. With the increase of the number of cores up until 4 Go quickly scales its throughput. However, after that the benefit of adding more cores becomes less visible. Scala with Akka, on the other hand, start slowly, but adding more cores results in significant increase of throughput when the number of process pairs also increase. Interestingly, Erlang performs great when the number of processes is low, but its throughput quickly degrades with the introduction of more processes to the system.

<figure>
	<img src="/images/non-blocked-16core.png" alt="16 Cores" style="width: 60%; margin-left: auto; margin-right: auto; display: block;"/>
	<figcaption style="text-align: center;">16 Cores</figcaption>
</figure>

<figure>
	<img src="/images/non-blocked-cores-erlang.png" alt="Erlang" style="width: 60%; margin-left: auto; margin-right: auto; display: block;"/>
	<figcaption style="text-align: center;">Erlang</figcaption>
</figure>

<figure>
	<img src="/images/non-blocked-cores-golang.png" alt="Go" style="width: 60%; margin-left: auto; margin-right: auto; display: block;"/>
	<figcaption style="text-align: center;">Go</figcaption>
</figure>

<figure>
	<img src="/images/non-blocked-cores-scala-akka.png" alt="Scla with Akka" style="width: 60%; margin-left: auto; margin-right: auto; display: block;"/>
	<figcaption style="text-align: center;">Scala with Akka</figcaption>
</figure>