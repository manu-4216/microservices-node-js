# Testing

Teacher code: https://github.com/StephenGrider/ticketing

We will use:

- jest
- supertest
- mongodb-memory-server

**Note**: _supertest does not send the cookies by default, the way the browsers do. Workaround:_

```js
import request from 'supertest';
import { app } from '../../app';

it('responds with details about the user when signed up', async () => {
  const authResponse = await request(app)
    .post('/api/users/signup')
    .send({
      email: 'test@test.com',
      password: 'password',
    })
    .expect(201);
  const cookie = authResponse.get('Set-Cookie');

  const response = await request(app)
    .get('/api/users/currentuser')
    .set('Cookie', cookie)
    .send()
    .expect(200);

  expect(response.body.currentUser.email).toEqual('test@test.com');
});
```

A more scalable solution is to have a helper function, and then call it in the test:

```js
it('...', async () => {
  const cookie = await signinHelper();

  // ...
});
```
