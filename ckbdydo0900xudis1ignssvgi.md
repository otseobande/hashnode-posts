## Fanning out messages from a RabbitMQ queue to another RabbitMQ server's exchange.

## The Problem

A few weeks ago, my team and I had a problem. We use a third-party service to power certain functionalities on our application and the service offered a RabbitMQ queue which we could subscribe to and listen for events. We connected our application to this queue and everything was fine until we noticed that some events were being received by our staging environment but not our production environment. We asked the third-party service if multiple environments could listen to the same events from the queue when it's published by the service. The customer support representative casually said, "You'll have to handle the data yourself, we provide just one queue per user account". I was sad for a few minutes and didn't know how to handle this situation because we needed the events to be sent to both our staging and production environments at the same time. I'll be using the terms "events" and "messages" interchangeably for the rest of the article as they mean the same thing in this context.

## The Solution

I asked my team if they had any ideas on how we might be able to handle this and we decided we needed to fan these events/messages out to these different environments when we got it from the third-party. The next question we needed to answer was how we were going to achieve this. I did some research and figured out we needed to publish these events to a [RabbitMQ Fanout Exchange](https://www.rabbitmq.com/tutorials/amqp-concepts.html#:~:text=A%20fanout%20exchange%20routes%20messages,the%20broadcast%20routing%20of%20messages.) and listen to this exchange on all our environments. This presented a new problem, we need to host our own RabbitMQ server. Find out how to do that [here](https://www.rabbitmq.com/download.html). We were able to quickly use an EC2 instance to spin up a server. Now that the server was up, we needed one more piece to complete this puzzle. We needed a middle man between the third-party queue and our own exchange. To do that, we created "The Bouncer". The bouncer listens to messages on the third-party RabbitMQ queue and publishes it to our RabbitMQ server's fanout exchange. Sounds pretty straight forward yeah? Let's take a look at how we achieved this.

The Bouncer was written with NodeJs using the [amqplib](https://www.npmjs.com/package/amqplib) package. The first thing we needed to do was to connect to both RabbitMQ servers.

```javascript
const amqp = require('amqplib');

const openThirdParty = amqp.connect({ ...third party credentials });
const openBouncer = amqp.connect({ ...hosted server credentials });
```

Next up, we needed to create a channel and a fanout exchange on that channel on our RabbitMQ server. To use `async/await` syntax we needed to wrap the remaining code in an async function.

```javascript
const startBouncer = async () => {
  const bouncerConnect = await openBouncer;
  const thirdPartyConnect = await openThirdParty;

  const bouncerChannel = await bouncerConnect.createChannel();
  const exchange = 'bounce_exchange';

  bouncerChannel.assertExchange(exchange, 'fanout', {
    durable: false
  });
}
```

The next part we needed to complete this was the code that listens to the third-party channel and bounces the message to the exchange we've created. Here it is:

```
... previous startBouncer code
  const publishMessages = (msg) => {
    if (msg) {
      const data = JSON.parse(msg.content.toString());
      const headerType = data.Header.Type;

      console.log(`Bouncing ${headerType} to consumers`);
      bouncerChannel.publish(exchange, '', Buffer.from(msg.content));
    }
  }

  const thirdPartyChannel = await thirdPartyConnect.createChannel();
  thirdPartyChannel.prefetch(0, false);
  thirdPartyChannel.consume(queue, publishMessages, { noAck: true });
}
```

That's it! We just needed to call the `startBouncer` function at the end of the script and everything worked fine. Let's take a look at the entire script.

```
const amqp = require('amqplib');

const openThirdParty = amqp.connect({ ...third party credentials });
const openBouncer = amqp.connect({ ...hosted server credentials });

const startBouncer = async () => {
  const bouncerConnect = await openBouncer;
  const thirdPartyConnect = await openThirdParty;

  const bouncerChannel = await bouncerConnect.createChannel();
  const exchange = 'bounce_exchange';

  bouncerChannel.assertExchange(exchange, 'fanout', {
    durable: false
  });

  const publishMessages = (msg) => {
    if (msg) {
      const data = JSON.parse(msg.content.toString());
      const headerType = data.Header.Type;

      console.log(`Bouncing ${headerType} to consumers`);
      bouncerChannel.publish(exchange, '', Buffer.from(msg.content));
    }
  }

  const thirdPartyChannel = await thirdPartyConnect.createChannel();
  thirdPartyChannel.prefetch(0, false);
  thirdPartyChannel.consume(queue, publishMessages, { noAck: true });
}

startBouncer();
```