# Cross-service data replication

## Reminder about the ticket "version"

Orders and tickets service both keep information about the ticket.

To keep them in sync, the orders service needs to listen for ticket events (created, updated).

**Reminder**: There is a _version_ in the _event_ payload, that we save in both ticket tables, to make sure we don't process an event twice, or out of order.

## Associate order-ticket

In MongoDB, we can use a feature called "ref/population", which is similar to keys in relational databases.

## Enum for order status

Different services will make use of the order status string. To avoid typos, we will create an enum, that we will share via the common library.

```ts
export enum OrderStatus {
  Created = 'created',
  Cancelled = 'cancelled',
  AwaitingPayment = 'awaitingPayment',
  Complete = 'complete',
}
```

## Defining the ticket model

Inside the order service, the order is composed of:

- userId
- status
- expiresAt
- ticketId => pointing at a ticket: { title, price, version }

Note that we don't want to reference ALL the ticket properties, but just the minimum ones. This allows us to keep the definition of ticket flexible (concert ticket, sport event ticket, etc).

## Adding a static method `isReserved` on the ticket model

This functionality is needed in multiple places.

Usage:

```js
const isReserved = await ticket.isReserved();
```

## Order events

Events published:

- order:created
- order:cancelled
