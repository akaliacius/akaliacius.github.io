---
layout: post
title:  "Vertx eventbus event types"
description: Reference to Vertx eventbus messaging
date:   2022-01-25 16:58:28 +0000
category: vertx
permalink: /:year/:month/:day/:title/
tags: Java Vertx
---

## Publish/Subscribe messaging
Messages published to an address.

![event](/assets/img/2022-02/eventbus.gif){:class="img-responsive" width="100%"}


Verticle publishes message every 5 seconds

```java

public class Publisher extends AbstractVerticle {
    @Override
    public void start(Promise<Void> startPromise) throws Exception {
      var message = new JsonObject()
        .put("greeting", "hello");
      vertx.setPeriodic(Duration.ofSeconds(5).toMillis(), id -> {
        logger.info("sending message: {}", message.encodePrettily());
        vertx.eventBus().publish("publisher.address", message);
      });
      startPromise.complete();
    }
}

```

Verticle subscribes and consumes messages

```java

public static class Subscriber1 extends AbstractVerticle {
    @Override
    public void start(Promise<Void> startPromise) throws Exception {
      startPromise.complete();
      vertx.eventBus().<JsonObject>consumer(
        "publisher.address",
        message -> logger.debug("Received: {}", message.body().encodePrettily())
      );
    }
}

```

## Point-to-point messaging

![event](/assets/img/2022-02/eventbus2.gif){:class="img-responsive" width="100%"}

## Request/Response messaging

![event](/assets/img/2022-02/eventbus2.gif){:class="img-responsive" width="100%"}