---
layout: post 
title:  Asynchronous HTTP requests with RxJava
description: Send long blocking requests asynchronously with RxJava and Vertx 
date:   2022-03-16 12:15:28 +0000 
category: rxjava
author: andrius kaliacius
permalink: /:year/:month/:day/:title/
tags: rxjava, vertx, async, java
---

## Overview

Let's say we develop a service that has to interact with other components. Unfortunately, those components are
slow and blocking.

It may be a legacy service that is very slow or some blocking API that we must use regardless we have no control 
over it. In this post, we will call 2 APIs. One of them will block for 2 seconds and another for 5 seconds. 

We also need to print
response status codes once both responses are available. If we do it in the old fashion, non-reactive way we 
would block a calling
thread for five seconds. Holding thread for five seconds is not efficient, is it?

![event](/assets/img/2022-03/requests.jpg){:class="img-responsive" width="100%"}



---
## Services

I used [httpstat.us](http://httpstat.us/) as a web service. This is a simple service for generating different 
HTTP codes to test web clients.
It is possible to provide extra parameters, in this case,
`sleep` that blocks HTTP requests for a provided amount of time.

Let's use [httpie](https://httpie.io/) to test both services 

Service 1 will block for 5 seconds and return a response with status code 200

```

http://httpstat.us/200?sleep=5000
_____________________________________________

HTTP/1.1 200 OK
Content-Length: 6
Content-Type: text/plain
Date: Tue, 08 Mar 2022 17:05:08 GMT
Request-Context: appId=cid-v1:1e93d241-20e4-4513-bbd7-f452a16a5d69
Server: Kestrel
Set-Cookie: ARRAffinity=e2c17206c539113795daf64bd958d003f2b29b9f62da53617beea05468875ba5;Path=/;HttpOnly;Domain=httpstat.us

200 OK

```

Service 2 is identical to the previous one except that it blocks for 2 seconds instead of 5.

```

http://httpstat.us/200?sleep=2000
_____________________________________________

HTTP/1.1 200 OK
Content-Length: 6
Content-Type: text/plain
Date: Tue, 08 Mar 2022 17:11:53 GMT
Request-Context: appId=cid-v1:1e93d241-20e4-4513-bbd7-f452a16a5d69
Server: Kestrel
Set-Cookie: ARRAffinity=e2c17206c539113795daf64bd958d003f2b29b9f62da53617beea05468875ba5;Path=/;HttpOnly;Domain=httpstat.us

200 OK

```



---
## Web Client

We have learned about services let's discuss web client. In this post, 
I used [Vert.x Web Client](https://vertx.io/docs/vertx-web-client/java/). It is asynchronous, 
easy to use HTTP and HTTP/2
client that supports RxJava too.

```java

private static Single<Integer> service1(WebClient webClient) {
        return webClient.getAbs("http://httpstat.us/200?sleep=5000")
                .rxSend()
                .doOnSuccess(response -> out.println("[" + Thread.currentThread().getName() + "] service 1: response received"))
                .map(HttpResponse::statusCode);
    }
}

private static Single<Integer> service2(WebClient webClient) {
        return webClient.getAbs("http://httpstat.us/200?sleep=2000")
                .rxSend()
                .doOnSuccess(response -> out.println("[" + Thread.currentThread().getName() + "] service 2 response received"))
                .map(HttpResponse::statusCode);
    }

```

Both methods are very similar. They take **WebClient** as parameter and send HTTP request 
returning **Single\<Integer\>**.
Where integer is an HTTP response code. Returning RxJava **Single** assures us that result is asynchronous. 
Status code will be
accessible later when it is available. This also gives us lazy evaluation, services will 
get invoked only if an active
subscription is present.

---
## Consuming Single sources

There are two sources that we will need to subscribe to. RxJava has a convenient method to combine
**Single** sources together. We can invoke method **.zipWith** on the first source and supply two 
parameters. The first is the source
to zip with and the second one is a function to consume both results, process them and return something else.

In this case, the return type is 
**AbstractMap.SimpleEntry\<Integer, Integer\>** that is a simple tuple of two integers. Looks verbose, doesn't it? 
Unfortunately, there are no better tuple or pair implementations in core Java libraries.

Thanks to Java lambdas we can pass behaviour as a parameter.

```java

Single<Integer> service1Code = service1(webClient);
Single<Integer> service2Code = service2(webClient);

Single<AbstractMap.SimpleEntry<Integer, Integer>> tupleSource = 
            service1Code.zipWith(service2Code, (s1, s2) -> new AbstractMap.SimpleEntry<>(s1, s2));

```

> You may implement your own tuple or pair if **AbstractMap.SimpleEntry** feels too verbose

---

## All together

Finally, we can put all bits and peaces together

```java
// Vertx instance and web client
Vertx vertx = Vertx.vertx();
WebClient webClient = WebClient.create(vertx);

// single sources. Lazy evaluation, no invocation at this point
Single<Integer> service1Code = service1(webClient);
Single<Integer> service2Code = service2(webClient);

// combine results together and create tuple
Single<AbstractMap.SimpleEntry<Integer, Integer>> tupleSource =
            service1Code.zipWith(service2Code, (s1, s2) -> new AbstractMap.SimpleEntry<>(s1, s2));

// subscribe and invoke services
tupleSource
    .doFinally(countDownLatch::countDown)
    .subscribe(Services::printResult);
    
```
Here are the results printed on the console after running the code. Both requests
were dispatched from the same vertx event loop thread.
The program also prints messages that the thread is not blocked every second. Finally, it prints both status 
codes as a final result. As you can see everything happened on the same thread.

```
[vert.x-eventloop-thread-1] is released
[vert.x-eventloop-thread-1] is released
[vert.x-eventloop-thread-1] service 2 response received
[vert.x-eventloop-thread-1] is released
[vert.x-eventloop-thread-1] is released
[vert.x-eventloop-thread-1] is released
[vert.x-eventloop-thread-1] service 1: response received
[vert.x-eventloop-thread-1] Result: service1:200 service2:200
```

---
That was all for this time. The complete code is on [gist](https://gist.github.com/akaliacius/1da6638f5df21fe83c13d055dd9ee679)

You can also run the code directly with [jbang](https://www.jbang.dev/)

`jbang https://gist.github.com/1da6638f5df21fe83c13d055dd9ee679`

Happy reactive coding!

