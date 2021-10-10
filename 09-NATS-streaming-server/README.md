# NATS streaming server

Next step: building the event-bus service.

Tackling this before other things (like building the orders service) is important for understanding some limitations of this pattern.

**Documentation**: docs.nats.io

NOTE: **NATS** is different than **NATS Streaming Server**
We will use **NATS Streaming Server** which is more advanced.

Docker image on docker hub: **nats-streaming**

## Creating a simple test NATS publisher and listener

First we will create a deployment and a service for kubernetes.

Then we will create a test NATS streaming publisher and listener.

```js
// package.json
"scripts": {
  "publish": "ts-node-dev --rs --notify false src/publisher.ts",
  "listen": "ts-node-dev --rs --notify false src/listener.ts"
}

// src/publisher.ts
import nats from 'node-nats-streaming';

const client = nats.connect('ticketing', 'abc', {
  url: 'http://localhost:4222'
});

client.on('connect', () => {
  const data = {
    id: [clientId],
    title: 'concert',
    price: 20,
  }

  const payload = JSON.stringify(data); // NATS only supports string data

  client.publish('ticket:created', data, () => {
    console.log('Event published');
  });
})


// src/listener.ts
import nats, { Message } from 'node-nats-streaming';

const client = nats.connect('ticketing', [clientID], {
  url: 'http://localhost:4222'
});

client.on('connect', () => {
  const subscription = client.subscribe('ticket:created', [queueGroupName]);
  subscription.on('message', (msg: Message) => {
    const data = msg.getData(); // this is a string or a buffer. But TS can't know which.
    if (typeof data === 'string') { // this check is only to make TS happy
      console.log(`Received event with data: ${data}`);
    }
  });
})
```

## How to test the NATS service with Kubernetes

For easy testing of a failure, use Kubernetes port forwarding. This will forward the host machine port directly to the NATS pod, without the need to add/remove it from the ingress-nginx, or adding a new nodePort service.

```
kubectl port-forward <pod_name> <host_port>:<pod_port>
```

## Avoiding handling a NATS event multiple times with NATS queue groups

We will have multiple NATS listeners for the same subscribers. This is because there will be multiple kubernetes pods with the same container. We want to avoid having an event sent to all listeners, to avoid the event being handled multiple times.

In NATS word, this can be avoided with **queue groups**. These need to be created inside a **channel**. This allows an event to be sent to only one of the subscribers, randomly, inside a specified queue group.

This is done with:

```js
const subscription = client.subscribe('ticket:created', [queueGroupName]);
```

## Avoiding lost events after errors with NATS subscriber options: manual ACK

This can be done with:

```js
const options = nats.subscriptionOptions().setManualAckMode(true); // this means that the event ACK will have to be sent back manually, which allows us to do that only after successful handling.
const subscription = client.subscribe(
  'ticket:created',
  [queueGroupName],
  options
);
```

Performing the manual ACK:

```js
  subscription.on('message', (msg: Message) => {
    ...
    msg.ack();
  });
```

## Graceful client shutdown

```js
// listener:
nats.on('close', () => {
  process.exit();
})

// close the channel before the process ends
process.on('SIGINT', nats.close))
process.on('SIGTERM', nats.close))
```

## Concurrency issues

- wrong event order
- double transactions

**Solutions:**

- run only one instance of the listener => not best
- ignore some issues, if not matters

**Better solution:**

1. use a "last transaction number" or "version" for each entity in the database
2. if receiving any wrong transaction number, drop it
3. after 'retry_time', the right transaction will be re-sent, and processed

This idea will become more evident later. And even more for other services.

The `version` will be dictated by the service corresponding to the entity. When an event is sent to another service, it will contain this `version`. These other services will make use of the version in their DB, to make sure they are processing them in the correct order.

## Event re-delivery

Use 2 options available in NATS `client.subscriptionOptions`:

- `setDeliverAllAvailable()` = sends all the past events when a service comes online
- `setDurableName([serviceName])` = sets a name for a durable listener, to avoid re-sending again to its queue group
