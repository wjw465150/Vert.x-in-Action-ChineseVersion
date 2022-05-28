# 第三章: 事件总线：Vert.x 应用程序的支柱

**This chapter covers**
  - What the event bus is
  - How to have point-to-point, request-reply, and publish/subscribe communications over the event bus
  - The distributed event bus for verticle-to-verticle communication across the network

The previous chapter introduced *verticles*. A Vert.x application is made up of one or more verticles, and each verticle forms a unit for processing asynchronous events. It is common to specialize verticles by functional and technical concerns, such as having one verticle for exposing an HTTP API and another for dealing with a data store. This design also encourages the deployment of several instances of a given verticle for scalability purposes.

What we have *not* covered yet is how verticles can communicate with each other. For example, an HTTP API verticle needs to *talk* to the data store verticle if the larger Vert.x application is to do anything useful.

Connecting verticles and making sure they can cooperate is the role of the *event bus*. This is important when building reactive applications—the event bus offers a way to transparently distribute event-processing work both inside a process and across several nodes over the network.

## 3.1 What is the event bus?

The event bus is a means for sending and receiving messages in an asynchronous fashion. Messages are sent to and retrieved from *destinations*. A destination is simply a freeform string, such as *incoming.purchase.orders* or *incoming-purchase-orders*, although the former format with dots is preferred.

Messages have a body, optional headers for storing metadata, and an expiration timestamp after which they will be discarded if they haven’t been processed yet.

Message bodies are commonly encoded using the Vert.x JSON representation. The advantage of using JSON is that it is a serialization format that can be easily transported over the network, and all programming languages understand it. It is also possible to use Java primitive and string types, especially as JVM languages that may be used for writing verticles have direct bindings for them. Last but not least, it is possible to register custom encoder/decoders (codecs) to support more specialized forms of message body serialization. For instance, you could write a codec for converting Java objects to a binary encoding of your own. It is rarely useful to do so, however, and JSON and string data cover most Vert.x applications’ needs.

The event bus allows for decoupling between verticles. There is no need for one verticle to access another verticle class—all that is needed is to agree on destination names and data representation. Another benefit is that since Vert.x is polyglot, the event bus allows verticles written in different languages to communicate with each other without requiring any complex language interoperability layer, whether for communications inside the same JVM process or across the network.

An interesting property of the event bus is that it can be extended outside of the application process. You will see in this chapter that the event bus also works across distributed members of a cluster. Later in this book you will see how to extend the event bus to embedded or external message brokers, to remote clients, and also to JavaScript applications running in a web browser.

Communications over the event bus follow three patterns:
  - Point-to-point messaging
  - Request-reply messaging
  - Publish/subscribe messaging

### 3.1.1 Is the event bus just another message broker?

Readers familiar with message-oriented middleware will have spotted the obvious resemblance between the event bus and a message broker. After all, the event bus exhibits familiar messaging patterns, such as the publish/subscribe pattern, which is popular for integrating distributed and heterogeneous applications.

The short answer is that no, the Vert.x event bus is not an alternative to Apache ActiveMQ, RabbitMQ, ZeroMQ, or Apache Kafka. The longer explanation is that it is an *event* bus for verticle-to-verticle communications inside an application, not a *message* bus for application-to-application communications. As you will see later in this book, Vert.x integrates with message brokers, but the event bus is no replacement for this type of middleware. Specifically, the event bus does not do the following:
  - Support message acknowledgments
  - Support message priorities
  - Support message durability to recover from crashes
  - Provide routing rules
  - Provide transformation rules (schema adaptation, scatter/gather, etc.)

The event bus simply carries *volatile* events that are being processed asynchronously by verticles.

Not all events are created equal, and while some may be lost, some may not. In our quest for writing *reactive applications*, you will see where to use data replication or message brokers such as Apache Kafka in combination with the event bus.1

The event bus is a simple and fast event conveyor, and we can take advantage of it for most verticle-to-verticle interactions, while turning to more costly middleware for events that cannot be lost.

>  **💡提示:** Readers familiar with messaging patterns may want to skim the next three subsections, or even skip them.

### 3.1.2 Point-to-point messaging

Messages are sent by producers to destinations, such as *a.b.c* in figure 3.1. Destination names are free-form strings, but the convention in the Vert.x community is to use separating dots. For example, we could use *datastore.new-purchase-order*s to send new purchase orders to be stored in a database.

With point-to-point messaging, one of the possibly multiple consumers picks a message and processes it. Figure 3.1 shows this with messages *M1*, *M2*, and *M3*.

![](Chapter3-EventBus.assets/Figure_3_1.png)

Messages are distributed in a round-robin fashion among the consumers, so they split message processing in equal proportions. This is why in figure 3.1 the first consumer processes *M1* and *M3*, while the second consumer processes *M2*. Note that there is no fairness mechanism to distribute fewer messages to an overloaded consumer.

### 3.1.3 Request-reply messaging

In Vert.x, the request-reply messaging communication pattern is a variation on point-to-point messaging. When a message is sent in point-to-point messaging, it is possible to register a *reply* handler. When you do, the event bus generates a temporary destination name dedicated solely to communications between the request message producer that is expecting a reply, and the consumer that will eventually receive and process the message.

This messaging pattern works well for mimicking remote procedure calls, but with the response being sent in an asynchronous fashion, so there is no need to keep waiting until it comes back. For example, an HTTP API verticle can send a request to a data store verticle to fetch some data, and the data store verticle eventually returns a reply message.

This pattern is illustrated in figure 3.2. When a message expects a reply, a reply destination is generated by the event bus and attached to the message before it reaches a consumer. You can inspect the reply destination name through the event-bus message API if you want, but you will rarely need to know the destination, since you will simply call a *reply* method on the message object. Of course, a message consumer needs to be programmed to provide a reply when this pattern is being used.

![](Chapter3-EventBus.assets/Figure_3_2.png)

### 3.1.4 Publish/subscribe messaging

In publish/subscribe communications, there is even more decoupling between producers and consumers. When a message is sent to a destination, all subscribers receive it, as illustrated by figure 3.3. Messages *M1*, *M2*, and *M3* are each sent by a different producer, and all subscribers receive the messages, unlike in the case of point-to-point messaging (see figure 3.1). It is not possible to specify reply handlers for publish/subscribe communications on the event bus.

Publish/subscribe is useful when you are not sure how many verticles and handlers will be interested in a particular event. If you need message consumers to get back to the entity that sent the event, go for request-reply. Otherwise, opting for point-to-point versus publish/subscribe is a matter of functional requirements, mostly whether all consumers should process an event or just one consumer should.

![](Chapter3-EventBus.assets/Figure_3_3.png)

## 3.2 The event bus in an example

Let’s put the event bus to use and see how we can communicate between independent verticles. The example that we’ll use involves several temperature sensors. Of course, we won’t use any hardware. Instead we’ll let temperatures evolve using pseudo-random numbers. We will also expose a simple web interface where temperatures and their average will be updated live.

A screenshot of the web interface is shown in figure 3.4. It displays the temperatures from four sensors and keeps their average up to date. The communication between the web interface and the server will happen using *server-sent events*, a simple yet effective protocol supported by most web browsers.

![](Chapter3-EventBus.assets/Figure_3_4.png)

![](Chapter3-EventBus.assets/Figure_3_5.png)

Figure 3.5 gives an overview of the application architecture. The figure shows two concurrent event communications annotated with ordering sequences *[1, 2, 3]* (a temperature update is being sent) and *[a, b, c, d]* (a temperature average computation is being requested).

The application is structured around four verticles:
  - *HeatSensor* generates temperature measures at non-fixed rates and publishes them to subscribers to the *sensor.updates* destination. Each verticle has a unique sensor identifier.
  - *Listener* monitors new temperature measures and logs them using SLF4J.
  - *SensorData* keeps a record of the latest observed values for each sensor. It also supports request-response communications: sending a message to *sensor.average* triggers a computation of the average based on the latest data, and the result is sent back as a response.
  - *HttpServer* exposes the HTTP server and serves the web interface. It pushes new values to its clients whenever a new temperature measurement has been observed, and it periodically asks for the current average and updates all the connected clients.

### 3.2.1 Heat sensor verticle

The following listing shows the implementation of the *HeatSensor* verticle class.

![](Chapter3-EventBus.assets/Listing_3_1.png)

The *HeatSensor* verticle class does not use any realistic temperature model but instead uses random increments or decrements. Hence, if you run it long enough, it may report absurd values, but this is not very important in our journey through reactive applications.

The event bus is accessed through the *Vertx* context and the *eventBus()* method. Since this verticle does not know what the published values will be used for, we use the *publish* method to send them to subscribers on the *sensor.updates* destination. We also use JSON to encode data, which is idiomatic with Vert.x.

Let’s now look at a verticle that consumes temperature updates.

### 3.2.2 Listener verticle

The following listing shows the implementation of the Listener verticle class.

![](Chapter3-EventBus.assets/Listing_3_2.png)

The purpose of the *Listener* verticle class is to log all temperature measures, so all it does is listen to messages received on the *sensor.updates* destination. Since the emitter in the *HeatSensor* class uses a publish/subscribe pattern, *Listener* is not the only verticle that can receive the messages.

We did not take advantage of message headers in this example, but it is possible to use them for any metadata that does not belong to the message body. A common header is that of an “action,” to help receivers know what the message is about. For instance, given a *database.operations* destination, we could use an action header to specify whether we intend to query the database, update an entry, store a new entry, or delete a previously stored one.

Let’s now look at another verticle that consumes temperature updates.

### 3.2.3 Sensor data verticle

The following listing shows the implementation of the SensorData verticle class.

 ![](Chapter3-EventBus.assets/Listing_3_3.png)

The *SensorData* class has two event-bus handlers: one for sensor updates and one for average temperature computation requests. In one case, it updates entries in a *HashMap*, and in the other case, it computes the average and responds to the message sender.

The next verticle is the HTTP server.

### 3.2.4 HTTP server verticle

The HTTP server is interesting as it requests temperature averages from the *SensorData* verticle via the event bus, and it implements the server-sent events protocol to consume temperature updates.

Let’s start with the backbone of this verticle implementation.

**SERVER IMPLEMENTATION**

The following listing shows a classical example of starting an HTTP server and declaring a request handler.

![](Chapter3-EventBus.assets/Listing_3_4.png)

The handler deals with three cases:
  - Serving the web application to browsers
  - Providing a resource for server-sent events
  - Responding with 404 errors for any other resource path

>  **💡提示:** Manually dispatching custom actions depending on the requested resource path and HTTP method is tedious. As you will see later, the *vertx-web* module provides a nicer *router* API for conveniently declaring handlers.

**THE WEB APPLICATION**

Let’s now see the client-side application, which is served by the HTTP server. The web application fits in a single HTML document shown in the following listing (I removed the irrelevant HTML portions, such as headers and footers).

![](Chapter3-EventBus.assets/Listing_3_5.png)

The JavaScript code in the preceding listing deals with server-sent events and reacts to update the displayed content. We could have used one of the many popular JavaScript frameworks, but sometimes it’s good to get back to basics.

>  **🏷注意:** You may have noticed that listing 3.5 uses a modern version of JavaScript, with arrow functions, no semicolons, and string templates. This code should work as is on any recent web browser. I tested it with Mozilla Firefox 63, Safari 12, and Google Chrome 70.

**SUPPORTING SERVER-SENT EVENTS**

Let’s now focus on how server-sent events work, and how they can be easily implemented with Vert.x.

Server-sent events are a very simple yet effective protocol for a server to push events to its clients. The protocol is text-based, and each event is a block with an event type and some data:

  ```
  event: foo
  data: bar
  ```
Each block event is separated by an empty line, so two successive events look like this:
  ```
  event: foo
  data: abc

  event: bar
  data: 123
  ```
Implementing server-sent events with Vert.x is very easy.

![](Chapter3-EventBus.assets/Listing_3_6.png)

Listing 3.6 provides the implementation of the *sse* method that deals with HTTP requests to the */sse* resource. It declares one consumer for each HTTP request for temperature updates, and it pushes new events. It also declares a periodic task to query the *SensorData* verticle and maintain the average in a request-reply manner.

Since these two handlers are for an HTTP request, we need to be able to stop them when the connection is lost. This may happen because a web browser tab is closed, or simply on page reloads. To do that, we obtain *stream* objects, and we declare a handler for each, just like we would with forms that accept callbacks. You will see in the next chapter how to deal with stream objects, and when they are useful.

We can also use a command-line tool, such as *HTTPie* or *curl*, against the running application to see the event stream, as in the following listing.

![](Chapter3-EventBus.assets/Listing_3_7.png)

> **☢警告:** At the time of writing, server-sent events are supported by all major web browsers except those from Microsoft. There are some JavaScript polyfills that provide the missing functionality to Microsoft’s browsers, albeit with some limitations.

### 3.2.5 Bootstrapping the application

Now that we have all the verticles ready, we can assemble them as a Vert.x application. The following listing shows a main class for bootstrapping the application. It deploys four sensor verticles and one instance of each other verticle.

![](Chapter3-EventBus.assets/Listing_3_8.png)

Running the main method of this class allows us to connect with a web browser to http://localhost:8080/. When you do, you should see a graphical interface similar to that in figure 3.4, with continuous live updates. The console logs will also display temperature updates.

## 3.3 Clustering and the distributed event bus

Our use of the event bus so far has been *local*: all communications happened within the same JVM process. What is even more interesting is to use Vert.x *clustering* and benefit from a *distributed* event bus.

### 3.3.1 Clustering in Vert.x

Vert.x applications can run in clustering mode where a set of Vert.x application nodes can work together over the network. They may be node instances of the same application and have the same set of deployed verticles, but this is not a requirement. Some nodes can have one set of verticles, while others have a different set.

Figure 3.6 shows an overview of Vert.x clustering. A *cluster manager* ensures nodes can exchange messages over the event bus, enabling the following set of functionalities:
  - Group membership and discovery allow discovering new nodes, maintaining the list of current nodes, and detecting when nodes disappear.
  - Shared data allows maps and counters to be maintained cluster-wide, so that all nodes share the same values. Distributed locks are useful for some forms of coordination between nodes.
  - Subscriber topology allows knowing what event-bus destinations each node has interest in. This is useful for efficiently dispatching messages over the distributed event bus. If one node has no consumer on destination *a.b.c*, there is no point in sending events from that destination to that node.

![](Chapter3-EventBus.assets/Figure_3_6.png)

There are several cluster manager implementations for Vert.x based on Hazelcast, Infinispan, Apache Ignite, and Apache ZooKeeper. Historically Hazelcast was the cluster manager for Vert.x, and then other engines were added. They all support the same Vert.x clustering abstractions for membership, shared data, and event-bus message passing. They are all functionally equivalent, so you will have to choose one depending on your needs and constraints. If you have no idea which one to pick, I recommend going with Hazelcast, which is a good default.

Finally, as shown in figure 3.6, the event-bus communications between nodes happen through direct TCP connections, using a custom protocol. When a node sends a message to a destination, it checks the subscriber topology with the cluster manager and dispatches the message to the nodes that have subscribers for that destination.

**What cluster manager should you use?**  

There is no good answer to the question of which cluster manager you should use. It depends on whether you need special integration with one library, and also on what type of environment you need to deploy. If, say, you need to use the Infinispan APIs in your code, and not just Infinispan as the cluster manager engine for Vert.x, you should go with Infinispan to cover both needs.

You should also consider your deployment environment. If you deploy to some environment where Apache ZooKeeper is being used, perhaps it would be a good choice to also rely on it for the Vert.x cluster manager.

By default, some cluster managers use multicast communications for node discovery, which may be disabled on some networks, especially those found in containerized environments like Kubernetes. In this case, you will need to configure the cluster manager to work in these environments.

As mentioned earlier, in case of doubt, choose Hazelcast, and check the project documentation for specific network configuration, like when deploying to Kubernetes. You can always change to another cluster manager implementation later.

### 3.3.2 From event bus to distributed event bus

Let’s get back to the heat sensor application that we developed earlier in this chapter. Moving to a distributed event bus is transparent for the verticles.

We will prepare two main classes with different verticle deployments, as illustrated in figure 3.7:
  - Four instances of *HeatSensor*, and one instance of *HttpServer* on port 8080
  - Four instances of *HeatSensor*, one instance of *Listener*, one instance of *SensorData*, and one instance of *HttpServer* on port 8081 (so you can run and test it on the same host)

The goal is to show that by launching one instance of each deployment in clustering mode, verticles communicate just as if they were running within the same JVM process. Connecting with a web browser to either of the instances will give the same view of the eight sensors’ data. Similarly, the *Listener* verticle on the second instance will get temperature updates from the first instance.

![](Chapter3-EventBus.assets/Figure_3_7.png)

We will use Infinispan as the cluster manager, but you can equally use another one. Supposing your project is built with Gradle, you’ll need to add vertx-infinispan as a dependency:

```groovy
implementation("io.vertx:vertx-infinispan:version")
```

The following listing shows the implementation of the main class *FirstInstance* that we can use to start one node that doesn’t deploy all of the application verticles.

![](Chapter3-EventBus.assets/Listing_3_9.png)

As you can see, starting an application in clustered mode requires calling the *clusteredVertx* method. The remainder is just classic verticle deployment.

The code of the second instance’s main method is very similar and is shown in the following listing.

![](Chapter3-EventBus.assets/Listing_3_10.png)

Both main classes can be run on the same host, and the two instances will discover each other. As before, you can start them from your IDE, or by running *gradle run -PmainClass=chapter3.cluster.FirstInstance* and *gradle run -PmainClass=chapter3.cluster.SecondInstance* in two different terminals.

>  **💡提示:** If you are using IPv6 and encountering issues, you can add the *-Djava.net.preferIPv4Stack=true* flag to the JVM parameters.

By default, the Vert.x Infinispan cluster manager is configured to perform discovery using network broadcast, so the two instances discover each other when they’re run on the same machine. You can also use two machines on the same network.

> **☢警告:** Network broadcast rarely works in cloud environments and many data centers. In these cases, the cluster manager needs to be configured to use other discovery and group membership protocols. In the case of Infinispan, the documentation has specific details at https://infinispan.org/documentation/.

Figure 3.8 shows the application running with one browser connected to the instance with port 8080 and another browser connected to the second instance with port 8081, and we see logs from the *Listener* verticle in the background. As you can see, both instances display events from the eight sensors, and the first instance has its average temperature updated so it can interact with the *SensorData* verticle on the second instance.

The distributed event bus is an interesting tool, as it is transparent to the verticles.

>  **💡提示:** The event-bus API has *localConsumer* methods for declaring message handlers that only work locally when running with clustering. For instance, a consumer for destination *a.b.c* will not receive messages sent to that destination from another instance in the cluster.

The next chapter discusses asynchronous data and event streams.

![](Chapter3-EventBus.assets/Figure_3_8.png)

## Summary
  - The event bus is the preferred way for verticles to communicate, and it uses asynchronous message passing.
  - The event bus implements both publish/subscribe (one-to-many) and point-to-point (many-to-one) communications.
  - While it looks like a traditional message broker, the event bus does not provide durability guarantees, so it must only be used for transient data.
  - Clustering allows networked instances to communicate over the distributed event bus in a transparent fashion, and to scale the workload across several application instances.

