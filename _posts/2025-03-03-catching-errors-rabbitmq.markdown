---
layout: post
title:  "RabbitMQ Exceptions Don't Always Get Caught"
date:   2025-03-03
categories: post
---
I was testing out a RabbitMQ connection when I realized that when a publish fails, exceptions are not thrown by default

The default max size of a RabbitMQ queue is 16MiB. I've set up a test to publish a message one byte bigger than that.

![RabbitMQ exceptions are not caught using default settings](/assets/images/2025-03-03-catching-errors-rabbitmq/not-caught.png)

The command window in the image is the output of the RabbitMQ server, which does show that the message failed to publish. Yet, an exception is not thrown. The method finishes fine.

Instead, when creating the channel, publisher confirmations and publisher confirmation tracking must be turned on for the publish to throw an exception.

```
IChannel channel = await connection.CreateChannelAsync(
    new CreateChannelOptions(
        publisherConfirmationsEnabled: true,
        publisherConfirmationTrackingEnabled: true));
```

They are set to false by default.

Now, with these settings enabled the exception is successfully thrown

![RabbitMQ exceptions are thrown with publisher confirmations](/assets/images/2025-03-03-catching-errors-rabbitmq/caught.png)

**RabbitMQ server version: 4.0.6**\
**RabbitMQ.Client version: 7.1.1**
