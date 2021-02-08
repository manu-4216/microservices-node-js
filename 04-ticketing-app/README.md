# Ticketing app

Teacher code: https://github.com/StephenGrider/ticketing

## Previous app + next steps

We saw a few advantages while developing the previous simple posts app:

- using async events via the event bus makes all the microservices self-sufficient
- setting up docker + kubernetes is a pain at start, but it makes things easier once that's done the first time.

But there are still some negative points that we'll have to handle:

- duplicated code
- hard to picture the flow of events between services
- hard to remember what properties an event should have
- hard to test some event flows
- kubernetes + all the services can be heavy to run on the computer
- event order assumptions: what if a comment event reached before the post has been created ?

Solutions to the issues listed above (same order):

- build a shared npm module to share the code across projects
- precisely define the events in this shared library
- write everything in typescript
- write tests for as much as possible
- run a k8s cluster in the cloud and develop on it almost as quickly as local
- introduce code to handle concurrency issues

## Ticketing app - features

- users can list a ticket for sale for an event
- other users can purchase this ticket
- any user can list tickets for sale and purchase it
- when a user attempts to buy an event, the ticket is locked for 15 minutes. The user has 15 minutes to enter their payment info
- while locked, no other user can purchase the ticket. After 15 minutes, the ticket should unlock
- ticket prices can be edited if they are not locked

## Tech stack overview

- production grade authentication
- real persistent storage
- stripe for payment
- cloud provider for kubernetes

## Data

- user
  - email
  - password
- ticket
  - title
  - price
  - userId
  - orderId
- order
  - userId
  - status (created | cancelled | awaitingPayment | completed)
  - ticketId
  - expiresAt
- charge
  - orderId
  - status (created | failed | completed)
  - amount
  - stripeId
  - stripRefundId

## Services

For this app, we will create one service per data type. This is not the only way, but it depends on the app.

- auth (handles user)
- tickets
- orders
- expiration (watches for orders, cancels them after 15 minutes)
- payments (cancels orders if payment fails, completes if succeeds)

## Events that will be used

- UserCreated UserUpdated
- TicketCreated TicketUpdated
- OrderCreated OrderCancelled OrderExpired
- ChargeCreated

## Tech stack

- client: `NextJS`, `TypeScript`
- services: `node`
- persistent storage: `mongoDB`
- storage for expiration service: `redis`
- event bus: `NATS streaming server`

## Auth service

- create a node ts server
- create a deployment + cluster IP service
- create Dockerfile => build image
- create skaffold config => auto change detection
- create ingress-nginx rules so that we can reach the service from the outside. Host: _ticketing.dev_
- update /etc/hosts for _ticketing.dev_ to point to localhost
- access _ticketing.dev/api/users/current_. There will be a warning in Chrome, because the ingress-nginx uses a fake certificate, and HSTS (only secured connections allowed). **Workaround**: type `thisisunsafe` anywhere on the Chrome tab, and the warning will disappear.
- _optional_: use a k8s cluster within a cloud provider (GC). For this: gcloud sdk > install context, GC > create cluster, GC > Cloud build, update skaffold config to point to GC build, run skaffold in the new context.
- create the routing in the express app

### Auth routes

| method | path               | body              | action              |
| ------ | ------------------ | ----------------- | ------------------- |
| POST   | /api/users/signup  | {email, password} | create account      |
| POST   | /api/users/signin  | {email, password} | sign in             |
| POST   | /api/users/signout | {}                | sign out            |
| GET    | /api/users/current | -                 | get info about user |

`express-validator` = express.js middleware to validate and sanitize the requests.

The error data structure from this `express-validator` library needs to be normalized. This will avoid inconsistencies across services which might use a different validation logic, and structure. This way, the react app will have a single consistent way of error structure.

### Consistent error handling

https://expressjs.com/en/guide/error-handling.html

```js
// Generic error - defined in TS as abstract (useful only for type)
class CustomError extends Error {
  statusCode = '';

  constructor(err) {
    super(err);
  }

  this.serializeErrors() {}
}

// Different implementations of CustomError:
class NotFoundError extends CustomError {
  statusCode = 400;

  constructor(err) {
    super('Route not found');
  }

  this.serializeErrors() {
    return [{ message: 'Not found' }]
  }
}

const errorHandler = (err, req, res, next) => {
  if (err instanceof CustomError) {
    return res.status(err.statusCode).send({ errors: err.serializeErrors() });
  }

  res.status(400).send({
    errors: [{ message: 'Something went wrong' }],
  });
};

app.use(errorHandler);

// In the code:
throw new RequestValidationError()
```

`express-async-errors` = package to avoid having to use next(error) in an async handler.

Sync function:

a) Sync handler function:

```js
app.all('*', (req, res) => {
  throw new Error('route not supported');
});
```

b) Async handler function:
**Without this package:**

```js
app.all('*', async (req, res, next) => {
  next(throw new Error('route not supported')); // if we don't call next, the promise will never be resolved.
});
```

**With this package:**

```js
app.all('*', async (req, res) => {
  throw new Error('route not supported'); // no need to call next anymore
});
```

### Authentication

Use JWT or a cookie to store the user auth data. This will then be used to authenticate users, without the need to re-query the auth service in each service.

Each service that needs to check if the user is authenticated will need to use the same logic of checking the token/cookie. This duplication can be shared, to avoid duplication.

**Issue**: we need to be able to revoke the user access:

- give the token an expiration time (eg 15 minutes)
- when received, check if the token is still valid.
- If expired:
  - option 1: call the auth service to get a refresh token
  - option 2: return an error, and let the client call the auth service

Another option for fixing this issue:

- have a short lived cache storing the list of banned users. This info only needs to live for 15 minutes (same as token lifespan).

**Another big issue** For SSR, the first request that the user does, cannot contain a JWT (only with a custom XHR done in JS this is possible). To fix this, we'll need to communicate our JWT inside a cookie, so that it will be sent in that first request.

_PS: This is not an issue for client-side rendered applications._

_PS: There is a workaround: service workers. But we won't use that._

_Remember: the JWT payload is not encrypted by default. But we can be sure it hasn't been tampered with. This is what makes it useful._

#### JWT token signing and reading

Once the JWT is created using a secret key, we need to share that secret key across all the services that will need to read the info it contains.

### Sharing secrets securely with Kubernetes

For this we can use a k8s feature called _secrets_.

**Step 1: create a secret**

Imperative way:

```bash
kubectl create secret generic [SECRET_SCOPE] --from-literal=[KEY]=[VALUE]
```

**Step 2: import a secret in the container as an env variable**

Inside the deployment yaml, add:

```yaml
spec:
  containers:
    - name: auth
      image: ...
      env:
        - name: [NAME_INSIDE_THE_CONTAINER]
          valueFrom:
            secretKeyRef:
              name: [SECRET_SCOPE]
              key: [KEY]
```

**Step 3: make use of the env variable inside the node.js code**

Just use it: `process.env.[ENV_VAR_NAME]`

Note: ideally any undefined environment variable should be detected as soon as the server start, and not at run time.

So before starting the server (`app.listen`) add a check of this type:

```js
if (!process.env[ENV_VAR_NAME]) {
  throw new Error('... should be defined');
}
```

**Typescript trick:** add an `!` after a variable if we want its warning ignored.

### User auth

Create a `currentUser` middleware, which will do:

**Signup handler:**

```js
router.post(
  '/api/users/signup',
  async (req: Request, res: Response) => {
    const { email, password } = req.body;

    // Check if user exists. If not, create one in the DB
    // (...)

    // Generate JWT
    const userJwt = jwt.sign(
      {
        id: user.id,
        email: user.email
      },
      process.env.JWT_KEY!
    );

    // Store the jwt in the session object (see `cookie-session`)
    req.session = {
      jwt: userJwt
    };
  })

```

**Signout route**

```js
router.post('/api/users/signout', (req, res) => {
  req.session = null;

  res.send({});
});
```

**Current user middleware**

```js
interface UserPayload {
  id: string;
  email: string;
}

// Add (augment) a new property to an existing interface
declare global {
  namespace Express {
    interface Request {
      currentUser?: UserPayload
    }
  }
}

import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';

export const currentUser = (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  if (!req.session?.jwt) {
    return next();
  }

  try {
    const payload = jwt.verify(
      req.session.jwt,
      process.env.JWT_KEY!
    ) as UserPayload;
    req.currentUser = payload;
  } catch (err) {}

  next();
};
```

**Current user router**

```js
import { currentUser } from '../middlewares/current-user';

router.get('/api/users/currentuser', currentUser, (req, res) => {
  res.send({ currentUser: req.currentUser || null });
});
```

**Auth middleware**

This will be used for any route requiring to be logged in.

```js
const requireAuth = (req, res, next) => {
  if (!req.currentUser) {
    return res.status(401).send();
  }

  next();
};
```
