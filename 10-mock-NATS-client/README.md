# Mock NATS client

## Reusable listener

We need to build many NATS streaming clients (listeners), all mostly the same setup.

So we need to build a sharable setup for listeners.

In Node.js world, we will use a npm package for this.

```js
// base-listener.ts
const createListener = ({ subject, queueGroupName, onMessage }) => {
  // ...

  return {
    client,
    listen,
    subscriptionOptions,
    parseMessage,
    ackWait,
  };
};

// Usage:
const someListener = createListener({
  subject: 'ticket:created',
  queueGroupName: 'payment-service',
  onMessage: (data, msg) => {
    console.log(data);
    msg.ack();
  },
});
```

## Typescript for strong typing data: generic

_NOTE: I adapted some of the code compared to the one explained in the course (use functions, not classes). And since I don't know much Typescript, the Typescript parts might not be valid anymore._

Goal: depending on the listener created (subject), know the `data` type.

```ts
// subjects.ts
export enum Subjects {
  TicketCreated = 'ticket:created',
  OrderUpdated = 'order:updated',
}

// events.ts
export interface TicketCreatedEvent {
  subject: Subject.TicketCreated;
  data: {
    id: string;
    title: string;
    price: number;
  };
}

// base-listener.ts
interface Event {
  subject: Subjects;
  data: any;
}

// Use a generic type
const createListener = ({ subject: T['subject'], queueGroupName: string, onMessage: (data: T['data'], msg: Message): void }): <T extends Event> => {
  // ...

  return {
    client,
    listen,
    subscriptionOptions,
    parseMessage,
    ackWait,
  };
};

// usage:
const someListener = createListener({
    subject: '...'
  queueGroupName: 'payment-service',
  onMessage: (data: TicketCreatedEvent['data'], msg: Message) => {
    console.log(data);
    msg.ack();
  },
}):<TicketCreatedEvent>;
```

## Reusable publisher

Same idea as the reusable listener, then put it in the shared npm module.

## Shared event name

This could be done in Typescript. But then, it would not be available to other backend languages.

Other solutions: use a cross language solution:

- JSON Schema
- Protobuf
