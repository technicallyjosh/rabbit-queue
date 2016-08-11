# Rabbit Queue

```
   ,---.     |    |    o|        ,---.
   |---',---.|---.|---..|---     |   |.   .,---..   .,---.
   |  \ ,---||   ||   |||        |   ||   ||---'|   ||---'
   `   ``---^`---'`---'``---'    `---\`---'`---'`---'`---'
```

## About

This module is based on [WRabbit](https://github.com/Workable/wrabbit)
and [jackrabbit](https://github.com/hunterloftis/jackrabbit).

It makes [RabbitMQ](http://www.rabbitmq.com/) managemnent and integration easy. Provides an abstraction layer
above [amqplib](https://github.com/squaremo/amqp.node).

It is written in typescript and requires node v6.0.0 or higher. If you want to use it
with an older version please dowload the sources and build them with target  (`/ts/tsconfig.json`).

## Connectint to [RabbitMQ](http://www.rabbitmq.com/)

```javascript
  const rabbit = new Rabbit(process.env.RABBIT_URL || 'amqp://localhost', {
    prefetch: 1, //default prefetch from queue
    replyPattern: true, //if reply pattern is enabled an exclusive queue is created
    logger: log4js.getLogger(`Rabbit-queue`),
    prefix: '' //prefix all queues with an application name
  });

  rabbit.on('connected', () => {
    //create queues, add halders etc.
  });

  rabbit.on('disconnected', (err = new Error('Rabbitmq Disconnected')) => {
    //handle disconnections and try to reconnect
    console.error(err);
    setTimeout(() => rabbit.reconnect(), 100);
  });
```

## Usage

```javascript
  const {Rabbit} = require('rabbit-queue');

  const rabbit = new Rabbit('amqp://localhost');

  rabbit.createQueue('queueName', { durable: false }, (msg, ack) => {
    console.log(msg.content.toString());
    ack('response');
  }).then(() => console.log('queue created'));

  rabbit.publish('queueName', { test: 'data' }, { correlationId: '1' })
    .then(() => console.log('message published'));

  rabbit.getReply('queueName', { test: 'data' }, { correlationId: '1' })
    .then((reply) => console.log('received reply', reply));

```

## Binding to topics


```javascript
  const {Rabbit} = require('rabbit-queue');
  const rabbit = new Rabbit('amqp://localhost');

  await rabbit.createQueue('queueName', { durable: false }, (msg, ack) => {
    console.log(msg.content.toString());
    ack('response');
  }).then(() => console.log('queue created'));

  await rabbit.bindToExchange('queueName', 'amq.topic', 'routingKey');
  //shortcut await rabbit.bindToTopic('queueName', 'routingKey');

  await rabbit.publishExchange('amq.topic', 'routingKey', { test: 'data' }, { correlationId: '1' })
    .then(() => console.log('message published'));
  //shortcut await rabbit.publishTopic( 'routingKey', { test: 'data' }, { correlationId: '1' });

```

## Advanced Usage with BaseQueueHandler (will add to dlq after 3 failed retries)

```javascript
  const rabbit = new Rabbit('amqp://localhost');

  class DemoHandler extends BaseQueueHandler {
    handle({msg, event, correlationId, startTime}) {
      console.log('Received: ', event);
      console.log('With correlation id: ' + correlationId);
    }
    afterDlq({msg, event}) {
      // Something to do after added to dlq
    }
  }

  new DemoHandler('demoQueue', rabbit,
    {
      retries: 3,
      retryDelay: 1000,
      logEnabled: true, //log queue processing time
      logger: log4js.getLogger('[demoQueue]') // logging in debug, warn and error.
    });

  rabbit.publish('demoQueue', { test: 'data' }, { correlationId: '4' });

```

### Get reply pattern implemented with a QueueHandler

```javascript
 const rabbit = new Rabbit(process.env.RABBIT_URL || 'amqp://localhost');
  class DemoHandler extends BaseQueueHandler {
    handle({msg, event, correlationId, startTime}) {
      console.log('Received: ', event);
      console.log('With correlation id: ' + correlationId);
      return Promise.resolve('reply'); // could be return 'reply';
    }

    afterDlq({msg, event}) {
      // Something to do after added to dlq
    }
  }

  new DemoHandler('demoQueue', rabbit,
    {
      retries: 3,
      retryDelay: 1000,
      logEnabled: true,
      logger: log4js.getLogger('[demoQueue]')
    });

  await rabbit.getReply('demoQueue', { test: 'data' }, { correlationId: '5' })
    .then((reply) => console.log('received reply', reply));
```

### Example where handle throws an error

```javascript
  const rabbit = new Rabbit(process.env.RABBIT_URL || 'amqp://localhost');
  class DemoHandler extends BaseQueueHandler {
    handle({msg, event, correlationId, startTime}) {
      return Promise.reject(new Error('test Error'));  // could be synchronous: throw new Error('test error');
    }

    afterDlq({msg, event}) {
      console.log('added to dlq');
    }
  }

  new DemoHandler('demoQueue', rabbit,
    {
      retries: 3,
      retryDelay: 1,
      logEnabled: true,
      logger: log4js.getLogger('[demoQueue]')
    });

  await rabbit.getReply('demoQueue', { test: 'data' }, { correlationId: '5' })
    .then((reply) => console.log('received reply', reply)); //reply will be '';
```

## Tests

The tests are set up with Docker + Docker-Compose,
so you don't need to install rabbitmq (or even node) to run them:

```npm run test-docker```