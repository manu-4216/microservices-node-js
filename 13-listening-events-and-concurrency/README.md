# Listening events and concurrency issues

## Listener implementation

```ts
export class TicketCreatedListener extends Listener<TicketCreatedEvent> {
  subject: Subjects.TicketCreated = Subjects.TicketCreated;
  queueGroupName = queueGroupName;

  onMessage(data: TicketCreatedEvent['data'], msg: Message) {
    const { id, title, price } = data;

    const ticket = Ticket.build({ id, title, price });
    await ticket.save();

    msg.ack();
  }
}
```

## Using the listener

```ts
// orders/src/index.ts
import { TicketCreatedListener } from '...';

new TicketCreatedListener(natsWrapper.client).listen();
```

## Optimistic concurrency control (OCC)

This feature can be used in MongoDB/Mongoose, but also other databases.

It allows us to use the _version_ of a record, in order to handle events in the correct order.

This will take care of updating the version number on multiple saves.

To use this feature, we will use an npm package, that gives the `version` awareness power to mongoose: `mongoose-update-if-current`.

PS: There is also an option to use a _timestamp_ instead of _version_.

**Test:**

```js
it('increments the version number on multiple saves', async () => {
  const ticket = Ticket.build({...});

  await ticket.save();
  expect(ticket.version).toEqual(0);
  await ticket.save();
  expect(ticket.version).toEqual(1);
})
```

## When to increment and include the version in an event ?

**Answer**: only in the primary (originating) service responsible for a record.
