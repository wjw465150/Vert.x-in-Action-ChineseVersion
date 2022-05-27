# 第二章: Vert.x 异步编程的基础知识

The first step toward building reactive systems is to adopt asynchronous programming. Traditional programming models based on blocking I/O do not scale as well as those that use non-blocking I/O. Serving more requests with fewer resources is very appealing, so where’s the catch? There is indeed a little problem: asynchronous programming is a non-trivial paradigm shift if you have never been exposed to it!  

The chapters in this part of the book will teach you the fundamental concepts of asynchronous programming by using the Vert.x toolkit. Thinking in
asynchronous operations is definitely approachable (and fun!) with Vert.x, and we will explore the main building blocks of a Vert.x application.  

**This chapter covers**
- What Vert.x is
- Why distributed systems cannot be avoided
- The challenges in programming resource-efficient networked applications
-  What asynchronous and non-blocking programming is
- What a reactive application is, and why asynchronous programming is not enough
- Alternatives to Vert.x

We developers live in an industry of buzzwords, technologies, and practices hype cycles. I have long taught university students the elements of designing, programming, integrating, and deploying applications, and I have witnessed first-hand how complicated it can be for newcomers to navigate the wild ocean of current technologies.

`Asynchronous` and `reactive` are important topics in modern applications, and my goal with this book is to help developers understand the core concepts behind these terms, gain practical experience, and recognize when there are benefits to these approaches. We will use Eclipse Vert.x, a toolkit for writing asynchronous applications that has the added benefit of providing solutions for the different definitions of what “reactive” means.

Ensuring that you understand the concepts is a priority for me in this book. While I want to give you a solid understanding of how to write Vert.x applications, I also want to make sure that you can translate the skills you learn here to other similar and possibly competing technologies, now or five years down the road.

## 1.1 Being distributed and networked is the norm

It was common 20 years ago to deploy business applications that could perform all operations while running isolated on a single machine. Such applications typically exhibited a graphical user interface, and they had local databases or custom file management for storing data. This is, of course, a gross exaggeration, as networking was already in use, and business applications could take advantage of database servers over the network, networked file storage, and various remote code operations.

Today, an application is more naturally exposed to end users through web and mobile interfaces. This naturally brings the network into play, and hence distributed sys- tems. Also, *service-oriented architectures* allow the reuse of some functionality by issuing requests to other services, possibly controlled by a third-party provider. Examples would be delegating authentication in a consumer application to popular account providers like Google, Facebook, or Twitter, or delegating payment processing to Stripe or PayPal.

## 1.2 Not living on an isolated island

Figure 1.1 is a fictional depiction of what a modern application is: a set of networked services interacting with each other. Here are some of these networked services:

  - A database like PostgreSQL or MongoDB stores data.
  - A search engine like Elasticsearch allows finding information that was previ- ously indexed, such as products in a catalog.
  - A durable storage service like Amazon S3 provides persistent and replicated data storage of documents.
  - A messaging service can be
    + An SMTP server to programmatically send emails.
    + A bot for interacting with users over messaging platforms, such as Slack, Tele- gram, or Facebook Messenger.
    + An integration messaging protocol for application-to-application integra- tion, like AMQP.
 - An identity management service like Keycloak provides authentication and role management for user and service interactions.

Monitoring with libraries like Micrometer exposes health statuses, metrics, and logs so that external orchestration tools can maintain proper quality of service, possibly by starting new service instances or killing existing ones when they fail.

![](Chapter1-Fundamentals.assets/Figure_1_1.png)

Later in this book you will see examples of typical services such as API endpoints, stream processors, and edge services. The preceding list is not exhaustive, of course, but the key point is that services rarely live in isolation, as they need to talk to other services over the network to function.

## 1.3 There is no free lunch on the network

The network is exactly where a number of things may go wrong in computing:

  - The bandwidth can fluctuate a lot, so data-intensive interactions between ser- vices may suffer. Not all services can enjoy fast bandwidth inside the same data center, and even so, it remains slower than communications between processes on the same machine.

  - The latency fluctuates a lot, and because services need to talk to services that talk to additional services to process a given request, all network-induced latency adds to the overall request-processing times.

  - Availability should not be taken for granted: Networks fail. Routers fail. Proxies fail. Sometimes someone runs into a network cable and disconnects it. When the network fails, a service that sends a request to another service may not be able to determine if it is the other service or the network that is down.

In essence, modern applications are made of distributed and networked services. They are accessed over networks that themselves introduce problems, and each ser- vice needs to maintain several incoming and outgoing connections.

## 1.4 The simplicity of blocking APIs

Services need to manage connections to other services and requesters. The traditional and widespread model for managing concurrent network connections is to allocate a thread for each connection. This is the model in many technologies, such as Servlets in Jakarta EE (before additions in version 3), Spring Framework (before additions in version 5), Ruby on Rails, Python Flask, and many more. This model has the advan- tage of simplicity, as it is *synchronous*.

Let’s look at an example where a TCP server echoes input text back to the client until it sees a /quit terminal input (shown in listing 1.3).

The server can be run using the Gradle run task from the book’s full example proj-

ect (`./gradlew run -PmainClass=chapter1.snippets.SynchronousEcho` in a termi- nal). By using the `netcat` command-line tool, we can send and receive text.

![](Chapter1-Fundamentals.assets/Listing_1_1.png)

> **TIP** You may need to install netcat (or nc) on your operating system.

On the server side, we can see the following trace.

![](Chapter1-Fundamentals.assets/Listing_1_2.png)

The code in the following listing provides the TCP server implementation. It is a clas- sical use of the `java.io` package that provides synchronous I/O APIs.

![](Chapter1-Fundamentals.assets/Listing_1_3.png)

```java
package chapter1.snippets;

import java.io.*;
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.net.Socket;

public class SynchronousEcho {

  public static void main(String[] args) throws Throwable {
    ServerSocket server = new ServerSocket();
    server.bind(new InetSocketAddress(3000));
    while (true) {   // <1>
      Socket socket = server.accept();
      new Thread(clientHandler(socket)).start();
    }
  }

  private static Runnable clientHandler(Socket socket) {
    return () -> {
      try (
        BufferedReader reader = new BufferedReader(
          new InputStreamReader(socket.getInputStream()));
        PrintWriter writer = new PrintWriter(
          new OutputStreamWriter(socket.getOutputStream()))) {
        String line = "";
        while (!"/quit".equals(line)) {
          line = reader.readLine();      // <2>
          System.out.println("~ " + line);
          writer.write(line + "\n");  // <3>
          writer.flush();
        }
      } catch (IOException e) {
        e.printStackTrace();
      }
    };
  }
}
```

><1>: 主应用程序线程扮演接受线程的角色，因为它接收所有新连接的套接字对象。 当没有连接挂起时，操作会阻塞。 为每个连接分配一个新线程。
>
><2>: 从套接字读取可能会阻塞分配给连接的线程，例如在读取的数据不足时。
>
><3>: 写入套接字也可能会阻塞，例如直到底层 TCP 缓冲区数据已通过网络发送。

The server uses the main thread for accepting connections, and each connection is allocated a new thread for processing I/O. The I/O operations are synchronous, so threads may block on I/O operations.

## 1.5 Blocking APIs waste resources, increase costs

The main problem with the code in listing 1.3 is that it allocates a new thread for each incoming connection, and threads are anything but cheap resources. A thread needs memory, and the more threads you have, the more you put pressure on the operating system kernel scheduler, as it needs to give CPU time to the threads. We could improve the code in listing 1.3 by using a thread pool to reuse threads after a connection has been closed, but we still need *n* threads for *n* connections at any given point in time.

This is illustrated in figure 1.2, where you can see the CPU usage over time of three threads for three concurrent network connections. Input/output operations such as `readLine` and `write` may *block* the thread, meaning that it is being parked by the oper- ating system. This happens for two reasons:
  - A read operation may be waiting for data to arrive from the network.
  - A write operation may have to wait for buffers to be drained if they are full from a previous write operation.

A modern operating system can properly deal with a few thousand concurrent threads. Not every networked service will face loads with so many concurrent requests,

![image-20220527153804390](Chapter1-Fundamentals.assets/Figure_1_2.png)

but this model quickly shows its limits when we are talking about tens of thousands of concurrent connections.

It is also important to recall that we often need more threads than incoming net- work connections. To take a concrete example, suppose that we have an HTTP service that offers the best price for a given product, which it does by requesting prices from four other HTTP services, as illustrated in figure 1.3. This type of service is often

![image-20220527154029412](Chapter1-Fundamentals.assets/Figure_1_3.png)

称为*边缘服务*或*API网关*。 按顺序请求每个服务然后选择最低价格会使我们的服务变得非常慢，因为每个请求都会增加我们自己服务的延迟。 有效的方法是从我们的服务启动四个并发请求，然后等待并收集它们的响应。 这意味着再启动四个线程； 如果我们有 1,000 个并发网络请求，我们可能会使用多达 5,000 个线程，在最糟糕的情况下，所有请求都需要同时处理，并且我们不使用线程池或维护来自边缘服务的持久连接 请求的服务。

Last, but not least, applications are often deployed to containerized or virtualized environments. This means that applications may not see all the available CPU cores, and their allocated CPU time may be limited. Available memory for processes may also be restricted, so having too many threads also eats into the memory budget. Such applications have to share CPU resources with other applications, so if all applications use blocking I/O APIs, there can quickly be too many threads to manage and sched- ule, which requires starting more server/container instances as traffic ramps up. This translates directly to increased operating costs.

## 1.6 Asynchronous programming with non-blocking I/O

Instead of waiting for I/O operations to complete, we can shift to *non-blocking* I/O. You may have already sampled this with the select function in C.

The idea behind non-blocking I/O is to request a (blocking) operation, and move

on to doing other tasks until the operation result is ready. For example a non-blocking read may ask for up to 256 bytes over a network socket, and the execution thread does other things (like dealing with another connection) until data has been put into the buffers, ready for consumption in memory. In this model, many concurrent connec- tions can be multiplexed on a single thread, as network latency typically exceeds the CPU time it takes to read incoming bytes.

Java has long had the java.nio (Java NIO) package, which offers non-blocking I/O APIs over files and networks. Going back to our previous example of a TCP ser- vice that echoes incoming data, listings 1.4 through 1.7 show a possible implementa- tion with Java non-blocking I/O.

![image-20220527154418444](Chapter1-Fundamentals.assets/Listing_1_4.png)

Listing 1.4 shows the server socket channel preparation code. It opens the server socket channel and makes it non-blocking, then registers an NIO key selector for pro- cessing events. The main loop iterates over the selector keys that have events ready for processing and dispatches them to specialized methods depending on the event type (new connections, data has arrived, or data can be sent again).

![image-20220527154620560](Chapter1-Fundamentals.assets/Listing_1_5.png)

Listing 1.5 shows how new TCP connections are dealt with. The socket channel that corresponds to the new connection is configured as non-blocking and then is tracked for further reference in a hash map, where it is associated to some *context object*. The context depends on the application and protocol. In our case, we track the current line and whether the connection is closing, and we maintain a connection-specific NIO buffer for reading and writing data.

![image-20220527154718266](Chapter1-Fundamentals.assets/Listing_1_6.png)

Listing 1.6 has the code for the echo method. The processing is very simple: we read data from the client socket, and then we attempt to write it back. If the write operation was only partial, we stop further reads, declare interest in knowing when the socket channel is writable again, and then ensure all data is written.

![image-20220527154921138](Chapter1-Fundamentals.assets/Listing_1_7.png)

Finally, listing 1.7 shows the methods for closing the TCP connection and for finishing writing a buffer. When all data has been written in continueEcho, we register interest again in reading data.

As this example shows, using non-blocking I/O is doable, but it significantly increases the code complexity compared to the initial version that used blocking APIs. The echo protocol needs two states for reading and writing back data: reading, or fin- ishing writing. For more elaborate TCP protocols, you can easily anticipate the need for more complicated state machines.

It is also important to note that like most JDK APIs, java.nio focuses solely on what it does (here, I/O APIs). It does not provide higher-level protocol-specific help- ers, like for writing HTTP clients and servers. Also, java.nio does not prescribe a threading model, which is still important to properly utilize CPU cores, nor does it handle asynchronous I/O events or articulate the application processing logic.

>  **🏷注意:** This is why, in practice, developers rarely deal with Java NIO. Network- ing libraries like Netty and Apache MINA solve the shortcomings of Java NIO, and many toolkits and frameworks are built on top of them. As you will soon discover, Eclipse Vert.x is one of them.

## 1.7 Multiplexing event-driven processing: The case of the event loop

A popular threading model for processing asynchronous events is that of the event loop. Instead of polling for events that may have arrived, as we did in the previous Java NIO example, events are pushed to an *event loop*.

![image-20220527155141864](Chapter1-Fundamentals.assets/Figure_1_4.png)

As you can see in figure 1.4, events are queued as they arrive. They can be I/O events, such as data being ready for consumption or a buffer having been fully written to a socket. They can also be any *other* event, such as a timer firing. A single thread is assigned to an event loop, and processing events shouldn’t perform any blocking or long-running operation. Otherwise, the thread blocks, defeating the purpose of using an event loop.

Event loops are quite popular: JavaScript code running in web browsers runs on top of an event loop. Many graphical interface toolkits, such as Java Swing, also have an event loop.

Implementing an event loop is easy.

![image-20220527155602740](Chapter1-Fundamentals.assets/Listing_1_8.png)

The code in listing 1.8 shows the use of an event-loop API whose execution gives the following console output.

![image-20220527155702905](Chapter1-Fundamentals.assets/Listing_1_9.png)

More sophisticated event-loop implementations are possible, but the one in the fol- lowing listing relies on a queue of events and a map of handlers.

![image-20220527155826507](Chapter1-Fundamentals.assets/Listing_1_10.png)

The event loop runs on the thread that calls the run method, and events can be safely sent from other threads using the `dispatch` method.

Last, but not least, an event is simply a pair of a key and data, as shown in the fol- lowing, which is a static inner class of `EventLoop`.

![image-20220527160000452](Chapter1-Fundamentals.assets/Listing_1_11.png)

## 1.8 What is a reactive system?

So far we have discussed how to do the following:
  - Leverage asynchronous programming and non-blocking I/O to handle more concurrent connections and use less threads
  - Use one threading model for asynchronous event processing (the event loop)

By combining these two techniques, we can build scalable and resource-efficient appli- cations. Let’s now discuss what a *reactive system* is and how it goes beyond “just” asyn- chronous programming.

The four properties of reactive systems are exposed in *The Reactive Manifesto*: *respon- sive*, *resilient*, *elastic*, and *message-driven* ([www.reactivemanifesto.org/](http://www.reactivemanifesto.org/)). We are not going to paraphrase the manifesto in this book, so here is a brief take on what these proper- ties are about:
  - *Elastic*—Elasticity is the ability for the application to work with a variable num- ber of instances. This is useful, as elasticity allows the app to respond to traffic spikes by starting new instances and load-balancing traffic across instances. This has an interesting impact on the code design, as shared state across instancesneeds to be well identified and limited (e.g., server-side web sessions). It is use- ful for instances to report *metrics*, so that an orchestrator can decide when to start or stop instances based on both network traffic and reported metrics.
  - *Resilient*—Resiliency is partially the flip side of elasticity. When one instance crashes in a group of elastic instances, resiliency is naturally achieved by redi- recting traffic to other instances, and a new instance can be started if necessary. That being said, there is more to resiliency. When an instance cannot fulfill a request due to some conditions, it still tries to answer in *degraded mode*. Depend- ing on the application domain, it may be possible to respond with older cached values, or even to respond with empty or default data. It may also be possible to forward a request to some other, non-error instance. In the worst case, an instance can respond with an error, but in a timely fashion.
  - *Responsive*—Responsivity is the result of combining elasticity and resiliency. Consistent response times provide strong service-level agreement guarantees. This is achieved both thanks to the ability to start new instances if need be (to keep response times acceptable), and also because instances still respond quickly when errors arise. It is important to note that responsivity is not possible if one component relies on a non-scalable resource, like a single central data- base. Indeed, starting more instances does not solve the problem if they all issue requests to one resource that is quickly going to be overloaded.
  - *Message-driven* —Using asynchronous message passing rather than blocking par- adigms like remote procedure calls is the key enabler of elasticity and resiliency, which lead to responsiveness. This also enables messages to be dispatched to more instances (making the system elastic) and controls the flow between mes- sage producers and message consumers (this is *back-pressure*, and we will explore it later in this book).

A reactive system exhibits these four properties, which make for dependable and resource-efficient systems.

**Does asynchronous imply reactive?**

This is an important question, as being asynchronous is often presented as being a magic cure for software woes. Clearly, reactive implies asynchronous, but the con- verse is not necessarily true.

As a (not so) fictitious example, consider a shopping web application where users can put items in a shopping cart. This is classically done by storing items in a server-side web session. When sessions are being stored in memory or in local files, the system is not reactive, even if it internally uses non-blocking I/O and asynchronous program- ming. Indeed, an instance of the application cannot take over another one because sessions are application state, and in this case that state is not being replicated and shared across nodes.

A reactive variant of this example would use a memory grid service (e.g., Hazelcast, Redis, or Infinispan) to store the web sessions, so that incoming requests could be routed to any instance.

## 1.9 What else does reactive mean?

As *reactive* is a trendy term, it is also being used for very different purposes. You just saw what a *reactive system* is, but there are two other popular reactive definitions, sum- marized in table 1.1.

**Table 1.1 All the reactive things**

| **Reactive?** | **Description**                                              |
| ------------- | ------------------------------------------------------------ |
| Systems       | Dependable applications that are message-driven, resilient, elastic, and responsive. |
| Programming   | A means of reacting to changes and events. Spreadsheet programs are a great example of reactive programming: when cell data changes, cells having formulas depending on affected cells are recomputed automatically. Later in this book you will see RxJava, a pop- ular *reactive extensions* API for Java that greatly helps coordinate asynchronous event and data processing. There is also *functional reactive programming*, a style of programming that we won’t cover in this book but for which *Functional Reactive Programming* by Stephen Blackheath and Anthony Jones (Manning, 2016) is a fantastic resource. |
| Streams       | When systems exchange continuous streams of data, the classical producer/consumer problems arise. It is especially important to provide *back-pressure* mechanisms so that a consumer can notify a producer when it is emitting too fast. With reactive streams |

## 1.10 What is Vert.x?

According to the Vert.x website (https://vertx.io/), “Eclipse Vert.x is a tool-kit for building reactive applications on the JVM.”

Initiated by Tim Fox in 2012, Vert.x is a project now fostered at the vendor-neutral Eclipse Foundation. While the first project iterations were aimed at being a “Node.js for the JVM,” Vert.x has since significantly deviated toward providing an asynchronous programming foundation tailored for the specifics of the JVM.

**The essence of Vert.x**

As you may have guessed from the previous sections of this chapter, the focus of Vert.x is processing asynchronous events, mostly coming from non-blocking I/O, and the threading model processes events in an event loop.

It is very important to understand that Vert.x is a *toolkit* and not a *framework*: it does not provide a predefined foundation for your application, so you are free to use Vert.x as a library inside a larger code base. Vert.x is largely unopinionated on the build tools that you should be using, how you want to structure your code, how you intend to package and deploy it, and so on. A Vert.x application is an assembly of modules pro- viding exactly what you need, and nothing more. If you don’t need to access a data- base, then your project does not need to depend on database-related APIs.

The Vert.x project is organized in composable modules, with figure 1.5 showing the structure of a random Vert.x application:
  - A core project, called vertx-core, provides the APIs for asynchronous pro- gramming, non-blocking I/O, streaming, and convenient access to networked protocols such as TCP, UDP, DNS, HTTP, or WebSockets.
  - A set of modules that are part of the community-supported Vert.x stack, such as a better web API (vertx-web) or data clients (vertx-kafka-client, vertx-redis, vertx-mongo, etc.) provide functionality for building all kinds of applications.
  - A wider ecosystem of projects provides even more functionality, such as con- necting with Apache Cassandra, non-blocking I/O to communicate between system processes, and so on.

![image-20220527160932294](Chapter1-Fundamentals.assets/Figure_1_5.png)

Vert.x is *polyglot* as it supports most of the popular JVM languages: JavaScript, Ruby, Kotlin, Scala, Groovy, and more. Interestingly, these languages are not just supported through their interoperability with Java. Idiomatic bindings are being generated, so you can write Vert.x code that still feels natural in these languages. For example, the Scala bindings use the Scala future APIs, and the Kotlin bindings leverage custom DSLs and functions with named parameters to simplify some code constructs. And, of course, you can mix and match different supported languages within the same Vert.x application.

## 1.11 Your first Vert.x application

It’s finally time for us to write a Vert.x application!

Let’s continue with the echo TCP protocol that we have used in various forms in this chapter. It will still expose a TCP server on port 3000, where any data is sent back to the client. We will add two other features:
  - The number of open connections will be displayed every five seconds.
  - An HTTP server on port 8080 will respond with a string giving the current num- ber of open connections.

### 1.11.1 Preparing the project

While not strictly necessary for this example, it is easier to use a build tool. In this book, I will show examples with Gradle, but you can find the equivalent Maven build descriptors in the book’s source code Git repository.

For this project, the only third-party dependency that we need is the vertx-core artifact plus its dependencies. This artifact is on Maven Central under the io.vertx group identifier.

An integrated development environment (IDE) like IntelliJ IDEA Community Edi- tion is great, and it knows how to create Maven and Gradle projects. You can equally use Eclipse, NetBeans, or even Visual Studio Code.

>  **TIP** You can also use the Vert.x starter web application at [https://start.vertx.io ](https://start.vertx.io/)and generate a project skeleton to download.

For this chapter let’s use Gradle. A suitable build.gradle.kts file would look like the next listing.

![image-20220527161309007](Chapter1-Fundamentals.assets/Listing_1_12.png)

> **TIP** You may be more familiar with Apache Maven than Gradle. This book uses Gradle because it is a modern, efficient, and flexible build tool. It also uses a concise domain-specific language for writing build files, which works better than Maven XML files in the context of a book. You will find Maven build descriptors equivalent to those of Gradle in the source code Git repository.

### 1.11.2 The VertxEcho class

The *VertxEcho* class implementation is shown in listing 1.15. You can run the applica- tion with Gradle using the run task (*gradle run* or *./gradlew run*), as follows.

![image-20220527161459706](Chapter1-Fundamentals.assets/Listing_1_13.png)

> **TIP** If you prefer Maven, run mvn compile exec:java instead of ./gradlew run from the chapter1 folder in the book’s source code Git repository.

You can, of course, interact with the service with the netcat command to echo text, and you can make an HTTP request to see the number of open connections, as shown in the following listing.

![image-20220527161603404](Chapter1-Fundamentals.assets/Listing_1_14.png)

>  **TIP** The http command comes from the HTTPie project at [https://](https://httpie.org/) [httpie.org](https://httpie.org/). This tool is a developer-friendly alternative to curl, and you can easily install it on your operating system.

Let’s now see the code of `VertxEcho`.

![image-20220527162403459](Chapter1-Fundamentals.assets/Listing_1_15.png)

```java
package chapter1.firstapp;

import io.vertx.core.Vertx;
import io.vertx.core.net.NetSocket;

public class VertxEcho {

  private static int numberOfConnections = 0;  // <1>

  public static void main(String[] args) {
    Vertx vertx = Vertx.vertx();

    vertx.createNetServer()
      .connectHandler(VertxEcho::handleNewClient)  // <2>
      .listen(3000);

    vertx.setPeriodic(5000, id -> System.out.println(howMany()));  // <3>

    vertx.createHttpServer()
      .requestHandler(request -> request.response().end(howMany()))  // <4>
      .listen(8080);
  }

  private static void handleNewClient(NetSocket socket) {
    numberOfConnections++;
    socket.handler(buffer -> {  // <5>
      socket.write(buffer);
      if (buffer.toString().endsWith("/quit\n")) {
        socket.close();
      }
    });
    socket.closeHandler(v -> numberOfConnections--);  // <6>
  }

  private static String howMany() {
    return "We now have " + numberOfConnections + " connections";
  }
}
```

> <1>: As you will see in the next chapter, event handlers are always executed on the same thread, so there is no need for JVM locks or using AtomicInteger.
>
> <2>: Creating a TCP server requires passing a callback for each new connection.
>
> <3>: This defines a periodic task with a callback being executed every five seconds.
>
> <4>: Similar to a TCP server, an HTTP server is configured by giving the callback to be executed for each HTTP request.
>
> <5>: The buffer handler is invoked every time a buffer is ready for consumption. Here we just write it back, and we use a convenient string conversion helper to look for a terminal command.
>
> <6>: Another event is when the connection closes. We decrement a connections counter that was incremented upon connection.

This example is interesting in that it has few lines of code. It is centered around a plain old Java main method, because there is no framework to bootstrap. All we need to create is a Vertx context, which in turns offers methods to create tasks, servers, cli- ents, and more, as you will discover in the next chapters.

While it’s not apparent here, an event loop is managing the processing of events, be it a new TCP connection, the arrival of a buffer, a new HTTP request, or a periodic task that is being fired. Also, every event handler is being executed on the same (event-loop) thread.

### 1.11.3 The role of callbacks

As you just saw in listing 1.15, *callbacks* are the primary method Vert.x uses to notify the application code of asynchronous events and pass them to some handlers. Combined with lambda expressions in Java, callbacks make for a concise way to define event handling.

You may have heard or experienced the infamous *callback hell* where callbacks get nested into callbacks, leading to code that is difficult to read and reason about.

![image-20220527163209979](Chapter1-Fundamentals.assets/Listing_1_16.png)

Be reassured: although the Vert.x core APIs indeed use callbacks, Vert.x provides sup- port for more programming models. Callbacks are the canonical means for notifica- tion in event-driven APIs, but as you will see in upcoming chapters, it is possible to build other abstractions on top of callbacks, such as futures and promises, reactive extensions, and coroutines.

While callbacks have their issues, there are many cases with minimal levels of nesting where they remain a very good programming model with minimal dispatch overhead.

### 1.11.4 So is this a reactive application?

This is a very good question to ask. It is important to remember that while Vert.x is a toolkit for building reactive applications, using the Vert.x API and modules does not “auto-magically” make an application a reactive one. Yet the event-driven, non-blocking APIs that Vert.x provides tick the first box.

The short answer is that no, this application is not reactive. Resiliency is not the issue, as the only errors that can arise are I/O related—and they simply result in dis- carding the connections. The application is also responsive, as it does not perform any complicated processing. If we benchmarked the TCP and HTTP servers, we would get very good latencies with low deviation and very few outliers. The following listing shows an imperfect, yet telling, quick benchmark with wrk ([https://github.com/](https://github.com/wg/wrk) [wg/wrk](https://github.com/wg/wrk)) running from a terminal.

![image-20220527163341291](Chapter1-Fundamentals.assets/Listing_1_17.png)

The culprit for not being reactive clearly is elasticity. Indeed, if we create new instances, each instance maintains its own connection counter. The counter scope is the application, so it should be a shared global counter between all instances.

As this example shows, designing reactive applications is more subtle than just implementing responsive and resource-efficient systems. Ensuring that an application can run as many replaceable instances is surprisingly more engaging, especially as we need to think about *instance state* versus *application state* to make sure that instances are interchangeable.

**What if I am a Windows user?**

wrk is a command-line tool that works on Unix systems like Linux and macOS.

In this book we prefer Unix-style tooling and command-line interfaces over graphical user interfaces. We will use Unix tools that are powerful, intuitive, and maintained by active open source communities.

Fortunately, you don’t have to leave Windows to benefit from these tools! While some of these tools work natively on Windows, starting from Windows 10 you can install the Windows Subsystem for Linux (WSL) and benefit from a genuine Linux environ- ment alongside your more traditional Windows desktop environment. Microsoft mar- kets WSL as a major feature for developers on Windows, and I can only recommend that you invest some time and get familiar with it. You can see Microsoft’s WSL FAQ for more details: https://docs.microsoft.com/en-us/windows/wsl/faq.

## 1.12 What are the alternatives to Vert.x?

As you will see in this book, Vert.x is a compelling technology for building end-to-end reactive applications. Reactive application development is a trendy topic, and it is more important to understand the principles than to blindly become an expert in one specific technology. What you will learn in this book easily transfers to other technolo- gies, and I highly encourage you to check them out.

Here are the most popular alternatives to Vert.x for asynchronous and reactive programming:
  - *Node.js*—Node.js is an event-driven runtime for writing asynchronous JavaScript applications. It is based on the V8 JavaScript engine that is used by Google Chrome. At first sight, Vert.x and Node.js have lots of similarities. Still, they dif- fer greatly. Vert.x runs multiple event loops by default, unlike Node.js. Also, the JVM has a better JIT compiler and garbage collector, so the JVM is better suited for long-running processes. Last, but not least, Vert.x supports JavaScript.
  - *Akka*—Akka is a faithful implementation of the *actor* model. It runs on the JVM and primarily offers Scala APIs, although Java bindings are also being pro- moted. Akka is particularly interesting, as actors are message driven and loca- tion transparent, and actors offer supervision features that are interesting for error recovery. Akka clearly targets the design of reactive applications. As you will see in this book, Vert.x is no less capable for the task. Vert.x has a concept of *verticles*, a loose form of actors, that are used for processing asynchronous events. Interestingly, Vert.x is significantly faster than Akka and most alter- natives in established benchmarks, such as TechEmpower benchmarks ([www.techempower.com/benchmarks/](www.techempower.com/benchmarks/))
  - *Spring Framework*—The older and widespread Spring Framework now integrates a reactive stack. It is based on Project Reactor, an API for reactive programming that is very similar to RxJava. The focus of the Spring reactive stack is essentially on reactive programming APIs, but it does not necessarily lead to end-to-end reac- tive applications. Many parts of the Spring Framework employ blocking APIs, so extra care must be taken to limit the exposure to blocking operations. Project Reactor is a compelling alternative to RxJava, but the Spring reactive stack is tied to this API, and it may not always be the best way to express certain asynchronous constructions. Vert.x provides more flexibility as it supports callbacks, futures, Java CompletionStage, Kotlin coroutines, RxJava, and fibers. This means that with Vert.x it is easier to select the right asynchronous programming model for a certain task. Also like with Akka, Vert.x remains significantly faster in TechEmpower benchmarks, and applications boot faster than Spring-based ones.
  - *Quarkus* —Quarkus is a new framework for developing Java applications that run exceptionally well in container environments like Kubernetes ([https://](https://quarkus.io/) [quarkus.io](https://quarkus.io/)). Indeed, in such environments, boot time and memory consumption are critical cost-saving factors. Quarkus employs techniques at compilation time to make sensible gains when running using traditional Java virtual machines and as native executables. It is based on popular libraries like Hibernate, Eclipse MicroProfile, RESTEasy, and Vert.x. Quarkus unifies imperative and reactive programming models, and Vert.x is a cornerstone of the framework. Vert.x is not just used to power some pieces of the networking stack; some client modules are directly based on those from Vert.x, such as the Quarkus mail service and reactive routes. You can also use Vert.x APIs in a Quarkus application, with the unification between reactive and imperative helping you to bridge both worlds. Vert.x and Quarkus have different programming paradigms: Vert.x will appeal to develop- ers who prefer a toolkit approach, or developers who have affinities with Node.js. In contrast, Quarkus will appeal to developers who prefer an opinionated stack approach with dependency injection and convention over configuration. In the end, both projects work together, and anything you develop with Vert.x can be reused in Quarkus.
  - *Netty* —The Netty framework provides non-blocking I/O APIs for the JVM. It provides abstractions and platform-specific bug fixes compared to using raw NIO APIs. It also provides threading models. The target of Netty is low-latency and high-performance network applications. While you can certainly build reactive applications with Netty, the APIs remain somewhat low-level. Vert.x is one of the many technologies built on top of Netty (Spring Reactive and Akka have Netty integration), and you can get all the performance benefits of Netty with the simpler APIs of Vert.x.
  - *Scripting languages* —Scripting languages such as Python and Ruby also provide non-blocking I/O libraries, such as Async (Ruby) and Twisted (Python). You can certainly build reactive systems with them. Again, the JVM performance is an advantage for Vert.x, along with the ability to use alternative JVM languages (Ruby is officially supported by Vert.x).
  - *Native languages*—Native languages are becoming trendy again. Instead of using the venerable C/C++ languages, Go, Rust, and Swift are gaining mindshare. They all tick the boxes for building highly scalable applications, and they certainly can be used for creating reactive applications. That being said, most efficient libraries in these languages are fairly low-level, and ultimately the JVM-based Vert.x/Netty combination still ranks favorably in benchmarks.

The following books are good resources for many of the preceding topics:
  - *Node.js in Action* by Mike Cantelon, Marc Harter, T.J. Holowaychuk, and Nathan Rajlich (Manning, 2013)
  - *Akka in Action* by Raymond Roestenburg, Rob Bakker, and Rob Williams (Man- ning, 2016)
  - *Reactive Application Development* by Duncan K. DeVore, Sean Walsh, and Brian Hanafee (Manning, 2018)
  - *Spring in Action*, fifth edition, by Craig Walls (Manning, 2018)
  - *Netty in Action* by Norman Maurer and Marvin Allen Wolfthal (Manning, 2015)
  - *Go in Action* by William Kennedy with Brian Ketelsen and Erik St. Martin (Man- ning, 2015)
  - *Rust in Action* by Tim McNamara (Manning, 2019)
  - *Swift in Depth* by Tjeerd in 't Veen (Manning, 2018)

In the next chapter, we will dissect the fundamentals of asynchronous programming with Vert.x.

## Summary

  - Asynchronous programming allows you to multiplex multiple networked con- nections on a single thread.
  - Managing non-blocking I/O is more complex than the equivalent imperative code based on blocking I/O, even for simple protocols.
  - The event loop and the reactor pattern simplify asynchronous event processing.
  - A reactive system is both scalable and resilient, producing responses with consis- tent latencies despite demanding workloads and failures.
  - Vert.x is an approachable, efficient toolkit for writing asynchronous and reac- tive applications on the JVM.

