# 第七章: Designing a reactive application

**This chapter covers**
  - What a reactive application is
  - Introducing the reactive application scenario used throughout part 2

The first part of this book taught you asynchronous programming with Vert.x. This is key to writing scalable and resource-efficient applications.

It is now time to explore what makes an application *reactive*, as we strive for both scalability and dependability. To do that, we will focus the following chapters on developing a fully reactive application out of several event-driven microservices. In this chapter, we’ll specify these services.

## 7.1 What makes an application reactive

In previous chapters we covered some elements of reactive:

◾     Back-pressure, as a necessary ingredient in asynchronous stream processing to regulate event throughput

◾     Reactive programming as a way to compose asynchronous operations

It is now time to explore the last facet: *reactive applications*. In chapter 1 I summarized *The Reactive Manifesto*, which declares that reactive applications are responsive, resilient, elastic, and message-driven.

The key property of reactive applications is that they remain responsive under demanding workloads and when they face the failure of other services. By “responsive,” we mean that latency for service response remains under control. A good responsiveness example would be a service that responds within 500 ms in the 99% percentile, provided that 500 ms is a good number given the service’s functional requirements and operational constraints.

An increasing workload will almost always degrade latency, but in the case of a reactive application, the goal is to avoid latency explosion when the service is under stress. Part 1 of this book mostly taught you asynchronous programming with Vert.x, which is the key ingredient for facing growing workloads. You saw that asynchronous processing of events allowed you to multiplex thousands of open network connections on a single thread. This model (when implemented correctly!) is much more resource-friendly and scalable than the traditional “1 thread per connection” model.

So Vert.x gives us a foundation for asynchronous programming on top of the JVM in order to meet demanding workloads, but what about dealing with failure? This is the other core challenge that we have to meet, and the answer is not a magic tool we can pick off the shelf. Suppose we have a service that talks to a database that becomes irresponsive because of an internal problem, like a deadlock. Some time will elapse before our service is notified of an error, perhaps in the form of a TCP connection timeout. In such a case, the latency explodes. By contrast, if the database is down, we get an immediate TCP connection error: latency is very good, but since the service cannot talk to its database, it is unable to process a request.

You will see in the last chapter of this part how to experiment with “what happens when things go wrong,” and we’ll discuss possible solutions for keeping services responsive. You might be tempted to enforce strict timeouts on all calls to other services (including databases), or to use *circuit-breakers* (more on that in the last chapter) everywhere, but a more analytical approach will help you see which solution to use, if any, and when. It is also important to see failure in the light of a service’s functional requirements and application domain: the response to failure may not always be an error. For instance, if we can’t get the latest temperature update from a sensor, we may serve the last known value and attach a timestamp to it, so the requester has all the necessary context attached to the data.

It is now time to build a reactive application, both to explore some elements of the Vert.x stack and to learn how to concretely build responsive applications.

## 7.2 The 10k steps challenge scenario

The application that we will implement in the upcoming chapters supports a (not so) fictional fitness-tracker challenge. Suppose we want to build an application to track and score users’ steps, as illustrated in figure 7.1.

![](Chapter7-DesigningAReactiveApplication.assets/Figure_7_1.png)

The application described in figure 7.1 would work as follows:

◾     Users sport connected pedometers that track how many steps they take.

◾     The pedometers regularly send step-count updates to the application that manages the challenge.

◾     The goal is to walk at least 10,000 steps each day, and users are greeted by an email every day when they do so.

◾     Users may choose to be publicly listed in rankings of step counts over the last 24 hours.

◾     Participants can also connect to a web application to see their data and update their information, such as their city and whether they want to appear in public rankings.

The web application allows new users to register by providing their device identifier as well as some basic information, such as their city and whether they intend to appear in public rankings (figure 7.2).

![](Chapter7-DesigningAReactiveApplication.assets/Figure_7_2.png)

Once connected, a user can update some basic details and get reminded of their total steps, monthly steps, and daily steps (figure 7.3).

![](Chapter7-DesigningAReactiveApplication.assets/Figure_7_3.png)

![](Chapter7-DesigningAReactiveApplication.assets/Figure_7_4.png)

There is also a separate web application that offers a public dashboard (figure 7.4).

The dashboard offers a ranking of public profiles over the last 24 hours, the current pedometer device update throughput, and trends by city. All the information displayed in the dashboard is updated live.

## 7.3 One application, many services

The application is decomposed as a set of (micro) services that interact with each other as in figure 7.5. Each service fulfills a single functional purpose and could well be used by another application. There are four public services: two user-facing web applications, one service for receiving pedometer device updates, and one service to expose a public HTTP API. The public API is used by the user web application, and we could similarly have mobile applications connect to it. There are four internal services: one to manage user profiles, one to manage activity data, one to congratulate users over email, and one to compute various stats over continuous events.

![](Chapter7-DesigningAReactiveApplication.assets/Figure_7_5.png)

> **NOTE** You may have heard of *command query responsibility segregation* (CQRS) and *event-sourcing*, which are patterns found in event-driven architectures.1 CQRS structures how to read and write information, while event sourcing is about materializing the application state as a sequence of facts. Our proposed application architecture relates to both notions, but because it’s not strictly faithful to the definitions, I prefer to just call it an “event-driven microservices architecture.”

All services are powered by Vert.x, and we also need some third-party middleware, labelled “infrastructure services” in figure 7.5. We’ll use two different types of databases: a document-oriented database (MongoDB) and a relational database (PostgreSQL). We need an SMTP server to send emails, and Apache Kafka is used for event-stream processing between some services. Because the ingestion service may receive updates from HTTP and AMQP, we’ll also use an ActiveMQ Artemis server.

There are two types of arrows in figure 7.5. Event flows show important event exchanges between services. For instance, the ingestion service sends events to Kafka, whereas the event stats service both consumes and produces Kafka events. I also denoted dependencies: for example, the public API service depends on the user profile and activities services, which in turn depend on their own databases for data persistence.

We can illustrate one example of interactions between services by looking at how a device update impacts the dashboard web application’s city trends ranking, as in figure 7.6.

![](Chapter7-DesigningAReactiveApplication.assets/Figure_7_6.png)

It all starts with a pedometer sending an update to the ingestion service, which verifies that the update contains all required data. The ingestion service then sends the update to a Kafka topic, and the pedometer device is acknowledged so it knows that the update has been received and will be processed. The update will be handled by multiple consumers listening on that particular Kafka topic, and among them is the activity service. This service will record the data to the PostgreSQL database and then publish another record to a Kafka topic with the number of steps recorded by the pedometer on that day. This record is picked up by the event stats service, which observes updates over windows of five seconds, splits them by city, and aggregates the number of steps. It then posts an update with the increment in steps observed for a given city as another Kafka record. This record is then consumed by the dashboard web application, which finally sends an update to all connected web browsers, which in turn update the display.

**About the application architecture**

As you dig through the specifications and implementations of the services, you may find the decomposition a bit artificial at times. For instance, the user profile and activity services could well be just one, saving some requests to join data from the two services. Remember that the decomposition was made for pedagogical reasons, and to show relevant elements from the Vert.x stack.

Making an application from (micro) services requires some compromises, especially as some services may be pre-existing, and you have to deal with them as they are, or you have limited ability to evolve them.

You may also find that the proposed architecture is not a nicely layered one, with some services nicely decoupled and some others having stronger dependencies on others. Again, this is done intentionally for pedagogical purposes. More often than not, real-world applications have to make compromises to deliver working software rather than pursue the quest for architectural perfection.

## 7.4 Service specifications

Let’s discuss the functional and technical specifications of the application services. For each service, we’ll consider the following elements:

◾     Functional overview

◾     API description, if any

◾     Technical points of interest, including crash recovery

◾     Scaling and deployment considerations

### 7.4.1 User profile service

The user profile service manages the profile data for a unique user. A user is identified by the following information:

◾     A username (must be unique)

◾     A password

◾     An email address

◾     A city

◾     A pedometer device identifier (must be unique)

◾     Whether the user wants to appear in public rankings or not

The service exposes an HTTP API and persists data in a MongoDB database (see figure 7.7).

![](Chapter7-DesigningAReactiveApplication.assets/Figure_7_7.png)

The service falls into the category of CRUD (for *create*, *read*, *update*, and *delete*) services that sit on top of a database. Table 7.1 identifies the different elements of the HTTP API.

This service is not to be publicly exposed; it is meant to be consumed by other services. There is no authentication mechanism in place. 

**Table 7.1 User profile HTTP API**

| **Purpose**                               | **Path**         | **Method** | **Data**                   | **Response**                                  | **Status code**                                              |
| ----------------------------------------- | ---------------- | ---------- | -------------------------- | --------------------------------------------- | ------------------------------------------------------------ |
| Register a new user                       | /register        | POST       | Registration JSON document | N/A                                           | 200 on success, 409 when the  username or device  identifier already exists, 500 for technical  errors |
| Get a user’s details                      | /<username>      | GET        | N/A                        | User data in JSON                             | 200 on success, 404 if the  username does not exist,  500 for technical errors |
| Update some user details                  | /<username>      | PUT        | User data in JSON          | N/A                                           | 200 on success, 500 for technical errors                     |
| Credentials validation                    | /authenticate    | POST       | Credentials in JSON        | N/A                                           | 200 on success, 401 when authentication fails                |
| Reverse lookup of a  user by their device | /owns/<deviceId> | GET        | N/A                        | JSON data with the username owning the device | 200 on success, 404 if the device  does not exist, 500  for technical errors |

The service is here to provide a facade for operations on top of the database. Both the service and the database can be scaled independently.

> **NOTE** The API described in table 7.1 does not follow the architectural principles of *representational state transfer* (REST) interfaces. A *RESTful* interface would expose user resources as, say, /user/<username>, and instead of registering new users through a POST request at /register, we would do so on the /user resource. Both faithful REST structures and more liberal HTTP API structures are valid choices.

### 7.4.2 Ingestion service

The ingestion service collects pedometer device updates and forwards records with update data to a Kafka stream for other services to process the events. The service receives device updates from either an HTTP API or an AMQP queue, as illustrated in figure 7.8. The service is a form of *protocol adapter* or *mediator*, as it converts events from one protocol (HTTP or AMQP) to another protocol (Kafka record streams).

A device update is a JSON document with the following entries:

◾     The device identifier

◾     A synchronization identifier, which is a monotonically increasing long integer that the device updates for each successful synchronization

◾     The number of steps since the last synchronization

![](Chapter7-DesigningAReactiveApplication.assets/Figure_7_8.png)

The HTTP API supports a single operation, as shown in table 7.2.

**Table 7.2 Ingestion service HTTP API**

| **Purpose**               | **Path** | **Method** | **Data**      | **Response** | **Status code**                           |
| ------------------------- | -------- | ---------- | ------------- | ------------ | ----------------------------------------- |
| Ingest a pedometer update | /ingest  | POST       | JSON document | N/A          | 200 on success, 500 for technical  errors |

The AMQP client receives messages from the step-events address. The JSON data is the same in both the HTTP API and AMQP client.

The service is meant to be publicly exposed so that it can receive pedometer updates. We assume that some reverse proxy will be used, offering encryption and access control. For instance, device updates over HTTPS could make use of client certificate checks to filter out unauthorized or unpatched devices.

AMQP and HTTP clients only get acknowledgements when records have been written to Kafka. In the case of HTTP, this means that a device cannot consider the synchronization to be successful until it has received an HTTP 200 response. The service does not check for duplicates, so it is safe for a device to consider the ingestion operation idempotent. As you will see, it is the role of the activity service to keep data consistent, and not that of the ingestion service.

The service can be scaled independently of the AMQP and the Kafka servers/ clusters. If the service crashes before some form of acknowledgement has been made, a client can always safely retry because of idempotency.

### 7.4.3 Activity service

The activity service keeps track of step-activity updates sent by the pedometers. The service stores events to a PostgreSQL database and offers an HTTP API to gather some statistics, such as daily, monthly, and total step counts for a given device. Updates are received from a Kakfa topic, which is fed by the ingestion service (see figure 7.9).

The activity service also publishes events with the number of steps for a device on the current day. This way, other services can subscribe to the corresponding Kafka topic and be notified rather than having to regularly poll the activity service for updates.

![](Chapter7-DesigningAReactiveApplication.assets/Figure_7_9.png)

The HTTP API is shown in table 7.3.

**Table 7.3 Activity service HTTP API**

| **Purpose**                                                  | **Path**                          | **Method** | **Data** | **Response**   | **Status code**                                              |
| ------------------------------------------------------------ | --------------------------------- | ---------- | -------- | -------------- | ------------------------------------------------------------ |
| Total step count  for a device                               | /<device id>/total                | GET        | N/A      | JSON  document | 200 on success, 404 if the device does  not exist, 500 for technical errors |
| Step count for a device in a particular month                | /<device id>/<year>/<month>       | GET        | N/A      | JSON  document | 200 on success, 404 if the device does  not exist, 500 for technical errors |
| Step count for a device on a particular  day                 | /<device id>/<year>/<month>/<day> | GET        | N/A      | JSON  document | 200 on success, 404 if the device does  not exist, 500 for technical errors |
| Ranking of the devices in decreasing number of steps over  the last 24 hours | /ranking-last-24-hours            | GET        | N/A      | JSON  document | 200 on success,  500 for technical errors                    |

Most of the operations are queries for a given device. As you will see in another chapter, the last operation provides an efficient query for getting a device’s ranking, which is useful when the dashboard service starts.

The events sent to the daily.step.updates Kafka topic contain the following information in a JSON document: 

◾     The device identifier

◾     A timestamp

◾     The number of steps recorded on the current day

For each incoming device update, there need to be three operations in this order:

◾     A database insert

◾     A database query to get the number of steps for the device on the current day

◾     A Kafka record write

Each of these operations may fail, and we don’t have a distributed transaction broker in place. We ensure idempotency and correctness as follows:

◾     We only acknowledge the incoming device update records in Kafka after the last operation has completed.

◾     The database schema enforces some uniqueness constraints on the events being stored, so the insertion operation can fail if an event is being processed again.

◾     We handle a duplicate insertion error as a normal case to have idempotency, and we follow along with the next steps until they have all completed.

◾     Successfully writing a daily steps update record to Kafka allows us to acknowledge the initial device update record, and the system can make progress with the other incoming records.

The activity service is not meant to be publicly exposed, so just like the user profile service, there is no authentication in place. It can be scaled independently of the database.

### 7.4.4 Public API

This service exposes a public HTTP API for other services to consume. It essentially acts as a *facade* over the user profile and activity services, as shown in figure 7.10.

![](Chapter7-DesigningAReactiveApplication.assets/Figure_7_10.png)

The service is a form of *edge service* or *API gateway* as it forwards and composes requests to other services. Since this is a public HTTP API, the service requires authentication for most of its operations. To do that we’ll use *JSON web tokens* ([https://tools](https://tools.ietf.org/html/rfc7519)

[.ietf.org/html/rfc7519](https://tools.ietf.org/html/rfc7519)), which we’ll discuss in chapter 8, along with the service implementation. Since we want the public API to be usable from any HTTP client, including JavaScript code running in a web browser, we need to support *cross-origin resource sharing*, or CORS (https://fetch.spec.whatwg.org/#http-cors-protocol). Again we will dig into the details in due time. The HTTP API operations are described in table 7.4.

**Table 7.4 Public API HTTP interface**

| **Purpose**                                             | **Path**                         | **Method** | **Data**                              | **Response**            | **Status code**                                   |
| ------------------------------------------------------- | -------------------------------- | ---------- | ------------------------------------- | ----------------------- | ------------------------------------------------- |
| Register a new user and device                          | /register                        | POST       | JSON document with registra tion data | N/A                     | 200 on success, 502  otherwise                    |
| Get a JWT token to use the API                          | /token                           | POST       | JSON document with credentials        | JWT token  (plain text) | 200 on success, 401  otherwise                    |
| Get a user’s data (requires a valid JWT)                | /<username>                      | GET        | N/A                                   | JSON  document          | 200 on success,  404 if not  found, 502 otherwise |
| Update a user’s data  (requires a valid JWT)            | /<username>                      | PUT        | JSON  document                        | N/A                     | 200 on success, 404 if not  found, 502 otherwise  |
| Total steps  of a user (requires a valid JWT)           | /<username>/total                | GET        | N/A                                   | JSON  document          | 200 on success,  404 if not  found, 502 otherwise |
| Total steps of a user in a month (requires a valid JWT) | /<username>/<year>/<month>       | GET        | N/A                                   | JSON  document          | 200 on success,  404 if not  found, 502 otherwise |
| Total steps of a user on a day (requires a valid JWT)   | /<username>/<year>/<month>/<day> | GET        | N/A                                   | JSON  document          | 200 on success,  404 if not  found, 502 otherwise |

Note that the request paths will be prefixed with /api/v1, so requesting a token is a POST request to /api/v1/token. It is always a good idea to have some versioning scheme in the URLs of a public API. The JWT tokens are restricted to the username that was used to obtain it, so user B cannot perform, say, a request to /api/v1/A/ 2019/07/14.

The public API service can be scaled to multiple instances. In a production setting, a load-balancing HTTP proxy should dispatch requests to the instances. There is no state to maintain in the service, since it forwards and composes requests to the other services.

### 7.4.5 User web application

The user web application provides a way for a user to register, update their details, and check some basic data about their activity. As shown in figure 7.11, there is a backend to serve the web application’s static assets to web browsers over HTTP.

![](Chapter7-DesigningAReactiveApplication.assets/Figure_7_11.png)

The frontend is a single-page application written in JavaScript and the Vue.JS framework. It is served by the user web application service, and all interactions with the application’s backend happen through calls to the public API service.

As such, this service is more a Vue.JS application than a Vert.x application, although it is still interesting to see how Vert.x serves static content with minimal effort. We could have chosen other popular JavaScript frameworks, or even no framework at all. I find Vue.JS to be a simple and efficient choice. Also, since Vue.JS embraces *reactive idioms*, it makes for a fully reactive application from the backend API to the frontend.

The service itself just serves static files, so it can be scaled to multiple instances and put behind a load balancer in a production setting. There is no state on the server side, either in the service or in the public API in use. It is the frontend application that stores some state in users’ web browsers.

### 7.4.6 Event stats service

The event stats service reacts to selected events from Kafka topics to produce statistics and publish them as Kafka records for other services to consume, as illustrated in figure 7.12.

![](Chapter7-DesigningAReactiveApplication.assets/Figure_7_12.png)

The service performs the following computations:

◾     Based on time windows of five seconds, it computes the throughput of device updates based on the number of events received on the incoming.steps topic, and it then emits a record to the event-stats.throughput topic.

◾     Events received on the daily.step.updates topic carry data on the number of steps from a device on the current day. This data lacks user data (name, city, etc.), so for each event the service queries the user profile service to enrich the original record with user data, and then sends it to the event-stats.useractivity.updates topic.

◾     The service computes city trends by processing the events from the eventstats.user-activity.updates topic over time windows of five seconds, and for each city it publishes an update with the aggregated number of steps for that city to the event-stats.city-trends.updates topic.

Kafka records can be acknowledged in an automatic fashion by batches, as there is little harm in processing a record again, especially for the throughput and city trends computations. To ensure that exactly one record is produced for an activity update, a manual acknowledgement is possible, although an occasional duplicate record should not impact a consuming service.

The event stats service is not meant to be public, and it does not offer any interface for other services. Finally, the service should be deployed as a single instance due to the nature of the computations.

### 7.4.7 Congrats service

The role of the congrats service is to monitor when a device reaches at least 10,000 steps on a day, and then to send a congratulation email to the owner, as shown in figure 7.13.

![](Chapter7-DesigningAReactiveApplication.assets/Figure_7_13.png)

The service makes calls to the user profile service to get the email of the user associated with a device, and then it contacts an SMTP server to send an email.

Note that we could have reused the event-stats.user-activity.updates Kafka topic fed by the event stats service, as it enriches the messages received from daily.step.updates with user data, including an email address. An implementation detail in how Kafka record keys are being produced for both topics makes it simpler to enforce that at most one message is sent to a user each day by using the records from daily.step.updates and then getting the email from the user profile service. This does not add much network and processing overhead either, since a user must receive an email only for the first activity update with at least 10,000 steps on a given day.

This service is not to be publicly exposed, and it does not expose any API. A single instance should suffice in a production setting, but the service can be scaled to multiple instances sharing the same Kafka consumer group so that they can split the workload among them.

### 7.4.8 Dashboard web application

The dashboard web application offers live updates on the incoming updates throughput, city trends, and public user ranking. As seen in figure 7.14, the service consumes Kafka records emitted by the event stats service and regularly pushes updates to the web application.

![](Chapter7-DesigningAReactiveApplication.assets/Figure_7_14.png)

The web application is written using the Vue.JS framework, just like the user web application described earlier. The frontend and backend are connected using the Vert.x event bus, so both the Vert.x and Vue.JS code bases can communicate with the same programming model.

Throughput and city trend updates from Kafka topics are directly forwarded over the Vert.x event bus, so the connected web application client receives the updates in real time. The backend maintains in-memory data about the number of steps over the last 24 hours for all users who have made their profile public. The ranking is updated every 5 seconds, and the result is pushed to the web application over the event bus so that the ranking is updated in the connected web browsers.

Since the backend is event-driven over Kafka topics, a good question is what happens when the service starts (or when it recovers from a crash). Indeed, on a fresh start we do not have all the step data from the last 24 hours, and we will only receive updates from the service’s start time.

We need a *hydration* phase when the service starts, where we query the activity service and get the rankings over the last 24 hours. We then need to query the user profile service for each entry of the ranking, since we need to correlate each device with a user profile. This is a potentially costly operation, but it shouldn’t happen very often.

Note that waiting for the hydration to complete does not prevent the processing of user activity updates, as eventually only the most recent value from either a Kafka record or the hydration data will prevail when updating the in-memory data.

The dashboard web application service is meant to be publicly exposed. It can be scaled to multiple instances if need be, and it can be put behind an HTTP proxy load balancer.

### 7.5 Running the application

To run the application, you need to run all the infrastructure services and all the microservices. The complete source code of the application can be found in the part2-steps-challenge folder of the source code repository.

First of all, Docker must be installed on your machine, because building the application requires containers to be started while executing test suites. The application can be built with Gradle using the gradle assemble command, or with gradle build if you also want to run the tests as part of the build and have Docker running.

Once the application services have been built, you will need to run all infrastructure services like PostgreSQL, MongoDB, Apache Kafka, and so on. You can greatly simplify the task by running them from Docker containers. To do that, the docker-compose.yml file describes several containers to be run with Docker Compose, a simple and effective tool for managing several containers at once. Running docker-compose up will start all the containers, and docker-compose down will stop and remove them all. You can also press Ctrl+C in a terminal running Docker Compose, and it will stop the containers (but not remove them, so they can be started with the current state again).

> **TIP** On macOS and Windows, I recommend installing Docker Desktop. Most Linux distributions offer Docker as a package. Note that docker needs to run as root, so on Linux you may need to add your user to a special group to avoid using sudo. The official Docker documentation provides troubleshooting instructions (https://docs.docker.com/engine/install/linux-postinstall/). In all cases, make sure that you can successfully run the docker run hello-world command as a user.

The container images that we will need to run are the following:

◾     MongoDB with an initialization script to prepare a collection and indexes

◾     PostgreSQL with an initialization script to create the schema

◾     Apache Kafka with Apache ZooKeeper from the Strimzi project images (see [https://strimzi.io](https://strimzi.io/))

◾     ActiveMQ Artemis

◾     MailHog, an SMTP server suitable for integration testing ([https://github.com/](https://github.com/mailhog/MailHog) [mailhog/MailHog](https://github.com/mailhog/MailHog))

All microservices are packaged as self-contained executable JAR files. For example, you can run the activity service as follows:

```
$ java -jar activity-service/build/libs/activity-service-all.jar
```

That being said, starting all services manually is not very convenient, so the project also contains a Procfile file to run all the services. The file contains lines with service names and associated shell commands. You can then use the Foreman tool to run the services (https://github.com/ddollar/foreman) or a compatible tool like Hivemind (https://github.com/DarthSim/hivemind):

```
$ foreman start
```

This is very convenient, as you can run all the services from two terminal panes, as illustrated in figure 7.15.

Foreman can also generate various system service descriptors from a Procfile: initab, launchd, systemd, and more. Finally, Foreman is written in Ruby, but there are also ports to other languages listed on the project page.

>  **TIP** Foreman simplifies running all services, but you don’t have to use it. You can run each individual service on the command line. The content of Procfile will show you the exact command for each service.

The next chapters will illustrate the challenges of implementing a reactive application by building on top of a set of (imperfect!) microservices that cover the topics of web, APIs, messaging, data, and continuous stream processing. In the next chapter, we’ll explore the web stack used to implement some of the services described in this chapter.

![](Chapter7-DesigningAReactiveApplication.assets/Figure_7_15.png)

## Summary

◾     A reactive application focuses on controlling latency under various workloads and in the presence of failures from other services.

◾     A reactive application can be decomposed as a set of independently scaled event-driven microservices.

