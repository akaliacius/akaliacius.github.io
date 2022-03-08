---
layout: post title:  "Asynchronous HTTP requests with RxJava"
description: Make long-lasting requests without blocking main thread date:   2022-03-08 15:15:28 +0000 category: cloud
permalink: /:year/:month/:day/:title/
---

Let's say we develop a service which has to interact with other components. Unfortunately, those components are
long-running. Maybe it's legacy service which is very slow, or some slow API that we have no control on, but must use
it. In this article we will call 2 API and one of blocks for 2 seconds and another for 5 seconds. We also need to print
response status codes once both responses are available. If we do it old fashion not reactive way we would block calling
thread for five seconds. Holding thread for five seconds is not efficient, is it?

## Services

I use [httpstat.us](http://httpstat.us/) as a web service. This is simple service for generating different HTTP codes.
It is possible to provide additional parameters, in this case,
`sleep` that blocks HTTP requests for provided amount of time.

To demonstrate [httpie](https://httpie.io/)

Service 1 will block for 5 seconds and return response with status code 200

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

Service 2 is identical to previous except that it blocks for 2 seconds instead of 5.

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

## Web Client

Okay, we have learned about services lets discuss web client. In this post, I
used [Vert.x Web Client](https://vertx.io/docs/vertx-web-client/java/). It is asynchronous, easy to use HTTP and HTTP/2
client that supports RxJava too.

```java

private static Single<Integer> service1(WebClient webClient) {
        return webClient.getAbs("http://httpstat.us/200?sleep=5000")
                .rxSend()
                .doOnSuccess(response -> out.println(Thread.currentThread().getName() + " service 1 invoked"))
                .map(HttpResponse::statusCode);
}

private static Single<Integer> service2(WebClient webClient) {
        return webClient.getAbs("http://httpstat.us/200?sleep=2000")
                .rxSend()
                .doOnSuccess(response -> out.println(Thread.currentThread().getName() + " service 2 invoked"))
                .map(HttpResponse::statusCode);
}

```

Both methods are very similar. They take **WebClient** as parameter and send HTTP request returning **Single<Integer>**.
Integer is HTTP response code. Returning RxJava **Single** assures us that this is asynchronous and status code will be
accessible later when it is available. This also gives us lazy evaluation, services will be invoked only if active
subscription is present.

## Consuming Single sources

There are two sources that we will need to subscribe to. RxJava has convenient method to combine
**Single** sources together. Simply call **.zipWith** on the first source and supply two parameters. First is the source
to zip with and the second one is a function to consume results and return something. In this case, return type is **
AbstractMap.SimpleEntry<Integer, Integer>** that is just a simple tuple of two integers. Thanks to Java lambdas we can
pass this function as parameter.

```java

Single<Integer> service1Code = service1(webClient);
Single<Integer> service2Code = service2(webClient);

Single<AbstractMap.SimpleEntry<Integer, Integer>> tupleSource = 
            service1Code.zipWith(service2Code, (s1, s2) -> new AbstractMap.SimpleEntry<>(s1, s2));

```

> You may implement your own tuple or pair if **AbstractMap.SimpleEntry** feels too verbose

## All together

Finally, we can see all bits and peaces together

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
That was all for this time. You can find complete code on [gist](https://gist.github.com/akaliacius/1da6638f5df21fe83c13d055dd9ee679)

Or, if you use [jbang](https://www.jbang.dev/) run it directly
`jbang https://gist.github.com/1da6638f5df21fe83c13d055dd9ee679`

Happy coding!

