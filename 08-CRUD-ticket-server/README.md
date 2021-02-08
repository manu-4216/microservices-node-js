# CRUD service setup

## Ticketing service overview

| method | path              | body                           | action          |
| ------ | ----------------- | ------------------------------ | --------------- |
| GET    | /api/tickets      | -                              | get all tickets |
| GET    | /api/tickets /:id | -                              | get one ticket  |
| POST   | /api/tickets      | {title: String, price: String} | create a ticket |
| PUT    | /api/tickets      | {title: String, price: String} | update a ticket |

The tickets service will have its own database (MongoDB).
Tickets collection: `{ title, price, userId }`

Folder structure:

```
-auth                         # auth service
  -src
  -Dockerfile
  -package.json
-tickets                      # tickets service
  -...
-client                       # NextJs app
-common                       # shared lib (npm package)
-infra                        # k8s, skaffold
  -k8s
    -auth-depl.yaml
    -auth-mongo-db-depl.yaml
    -client-depl.yaml
    -ingress-srv.yaml
    -ticket-depl.yaml
```

Then:

- build and push the `tickets` image
- write k8s for deployment and service
- update skaffold
- write k8s for mongodb deployment and service

Also:

- replace the hardcoded DB connection URI with environment variables, for both the existing `auth` and the new `tickets` service.

## Testing

We can use TDD to build this new service.

So we can start by building tests which interact with the new `tickets` service.

### Faking the auth cookie

However, note that some endpoints require the user to be authenticated.

**However, we have one problem: how can we detect, within the tests, if the user is authenticated or not?**

We can't call `auth` service `/api/signup` endpoint, since we don't want to introduce such a dependency in the test.

**Best solution:** fake the signin state. **How?**

What does the signin actually do ? **It sets a JWT session cookie.**

So we simply needs to manually create this cookie.

Here are the steps:

```ts
// jest setup
import jwt from 'jsonwebtoken';

declare global {
  namespace NodeJS {
    interface Global {
      signin(): string[];
    }
  }
}

global.signin = () => {
  // Build a JWT payload (id, email)
  const payload = {
    id: '123',
    email: 'test@test.com',
  };
  // Create the JWT
  const token = jwt.sign(payload, process.env.JWT_KEY);
  // Build the session object { jwt: MY_JWT }
  const session = { jwt: token };
  // Turn it into JSON
  const sessionJSON = JSON.stringify(session);
  // Encode it as base64
  const base64 = Buffer.from(sessionJSON).toString('base64');
  // Create the string with the final cookie matching the format outputted by `cookie-session`
  const cookieString = `express:sess=${base64}`;
  // request needs the cookie to be an array
  return [cookieString];
};

// Using it in a test:
it('can access /api/tickets only if authenticated', () => {
  const response = await request(api)
    .post('/api/tickets')
    .set('Cookie', global.signin());

  expect(response.status).not.toEqual(401);
});
```

### Implementing a route

Just as an example:

```js
import express, { Request, Response } from 'express';
import { requireAuth } from '@common';

const router = express.Router();

router.post(
  '/api/tickets',
  requireAuth,
  [
    // validation middleware: title, price
  ],
  (req: Request, res: Response) => {
    res.sendStatus(200);
  }
);
```

### Saving to a DB

Test:

```js
import { Ticket } from '../models/ticket';

it('creates a ticket', async () => {
  let tickets = await Ticket.find({});
  expect(tickets.length).toEqual(0);

  await request(app)
    .post('/api/tickets')
    .set('Cookie', global.signin())
    .set({
      title: 'abc',
      price: 20,
    })
    .expect(201);

  tickets = await Ticket.find({});
  expect(tickets.length).toEqual(1);
  expect(tickets[0].title).toEqual('abc');
});
```

Implementation:

```js
router.post(
  '/api/tickets',
  requireAuth,
  [
    // validation middleware: title, price
  ],
  async (req: Request, res: Response) => {
    const { title, price } = req.body;
    const ticket = Ticket.build({
      title,
      price,
      userId: req.currentUser!.id,
    });

    await ticket.save();

    res.status(201).send(ticket);
  }
);
```

Similar thing for the other tickets endpoints.
