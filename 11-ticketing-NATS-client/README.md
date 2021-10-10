# Ticketing NATS client

## Publishing events

```js
// Inside the request handler:
await ticket.save();

// After we save the ticket in the DB, emit an event
// QUESTION: do we need to `await` for the event ?
new TicketCreatedPublisher(natsWrapper.client).publish({
  id: ticket.id,
  title: ticket.title,
  price: ticket.price,
  userId: ticket.userId,
});

res.sent(ticket);
```

## Data integrity: saving the ticket event locally

We don't need to await for the event to be sent successfully. Instead:

- save the event in a database, owned by the ticket service, with its status "sent"="NO"
- then asynchronously, send the event to the message queue service
- once the event sending is successful, update the database to "sent"="YES"

Advantages:

- no data integrity issues across services
- we can resolve the request sooner, without an extra await

## Data integrity: use a transaction for both ticket save and event save

This would ensure that, if there is an error in the middle, the whole transaction would get dropped(rolled back).
