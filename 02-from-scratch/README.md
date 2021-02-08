# Purpose

Build a mini-microservice app, from scratch.
Blog-like app, to handle posts, and comments.

## Entities:

- posts
- comments

## Responsibilities:

Posts service:

- create a post
- list all posts

Comments service:

- create a comment
- list all comments

## Tech stack

_Client_: React app
_Backend_: Express applications, for each service
_Data persistance_: store in memory, to make it simple at start

### Posts service details

| method | path   | body              | action            |
| ------ | ------ | ----------------- | ----------------- |
| GET    | /posts | -                 | list all posts    |
| POST   | /posts | { title: string } | create a new post |

Posts stored as: { id<postId>, title }

### Comments service details

| method | path                | body                | action                             |
| ------ | ------------------- | ------------------- | ---------------------------------- |
| GET    | /posts/:id/comments | -                   | list all comments for a post id    |
| POST   | /posts/:id/comments | { content: string } | create a new comment for a post id |

Comments stored as: { id<postId>: [{ id<commentId>, content }] }

### Improvement on data model & query

If we want to get all the comments, now we need to make many calls.

One solution for this is to use a _query service_, responsible of combining the data from posts and comments, and have them ready.

This service would consume:

- requests for posts:

  | method | path   | body | action                              |
  | ------ | ------ | ---- | ----------------------------------- |
  | GET    | /posts | -    | list all comments for all the posts |

Posts with comments stored as:

```js
{ id<postId>: {
    id<postId>,
    title,
    comments: [{ id<commentId>, content }]
  }
}
```

- events for post and comment creation

### Event bus implementation

This is a service which listen to events, and then sends them forward to other services.

One simple implementation:

_PART 1_. Server for event-sending services (post, comments services)

| method | path    | body         | action                                           |
| ------ | ------- | ------------ | ------------------------------------------------ |
| POST   | /events | {type, data} | handles event received from the emitting service |

_PART 2_. Client for other event-listening services (query service)

So these listening services will have to listen to the forwarded events with their own route dedicated for this purpose:

| method | path    | body         | action                                    |
| ------ | ------- | ------------ | ----------------------------------------- |
| POST   | /events | {type, data} | handles forwarded event via the event bus |

### Consuming the forwarded events in the query service

```js
app.post('/events', (req, res) => {
  const { type, data } = req.body;

  if (type === 'PostCreated') {
    const { id, title } = data;
    posts[id] = { id, title, comments: [] };
  }

  if (type === 'CommentCreated') {
    const { id, postId, content } = data;
    posts[postId].comments.push({ id, content });
  }

  res.send({});
});
```

### New simple feature: comment moderation (async task)

Approach:

- add a new `status` (pending, approved, rejected) to the comments
- the query service will still listen to `CommentCreated` event
- the React frontend will adapt the UI, depending on the comment `status`
- add new _moderation service_, which will:
  - listen to `CommentCreated` events, so it knows when to trigger the moderation
  - once the moderation is done, emit `CommentModerated` event -> listened to by the _Comments service_
- after the comment status is updated (pending -> approved/rejected), emit `CommentUpdated` event
- this will then be picked up by the query service, to update the comment status

This approach keeps the responsibilities separated:

1. the comments service is responsible of knowing the comment structure, and update logic
2. the moderation service is independent, and can evolve on its own, rather than integrated in the comment service

### Issue to fix: out of sync events

_Example 1_
If the moderation service was temporarily down, then the previously sent events were never processed. So the application state will be out-of-sync. How can we solve this ?

_Example 2_
Query service is new, and it needs to be brought in sync with the current state of the application. For instance, we might have years of data that never send any events. So we need to have a way to initialize the DB of this new service.

#### Solution

Have the event bus store all the past events. Then, when a service needs to be synchronized, it will read all the missed events until the current time.

PS: Optimization: only ask the missed events, which happened after the last one that was processed. This means that we'd have to persist the last event, and pass it to the event bus.

**Note: The implementation bellow is just a oversimplified version of what a production event bus does.**

```js
// Event bus service
const events = []; // used for persistance

app.post('/events', (req, res) => {
  ...
  events.push(event);
})

app.get('/events', (req, res) => {
  res.send(events);
})

// Query service
app.post('/events', (req, res) => {
  const { type, data } = req.body;

  // extract function
  handleEvent(type, data);

  res.send({});
});

app.listen(<port>, async () => {
  console.log('listening to port <port>');

  // sync all missed events
  const { data: events } = await axios.get('<eventBusServiceUrl>/events');

  events.forEach(({ type, data }) => {
    handleEvent(type, data);
  })
})
```
