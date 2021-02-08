# What are microservices

This is best understood in comparison with a monolith architecture, in which there is a single unit, containing: routing, middleware, business logic, and database access _to implement ALL FEATURES of the app_.

In comparison, a _single_ microservice contains the same parts, but these are used to implement _one single feature_.

## Types of communication

1. SYNC: direct communication between services (eg.HTTP request)

Plus:

- concept simple to understand
- no need for a separate DB for this service, as it relies on the data from other services

Minus:

- adds dependency across services
- overall request is as fast as the slowest step
- spaghetti code

2. ASYNC: indirect communication, using an _event bus_

### OPTION 1: Standard use of events, via the event bus

```js
event: {
  type: '...',
  data: '...'
}
```

Same plus and minuses as the ASYNC.

### OPTION 2: Services make local changes to their own domain. And will also send events to the event bus. Other services can react to that event to build/update their own data, as these events happen.

Plus:

- Service has zero dependencies on other services
- Service will be very fast, because its data is already stored in its DB

Minus:

- Some data duplication (separate DB, built at event reception)
- Harder to understand
