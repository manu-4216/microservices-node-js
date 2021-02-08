# SSR React Next.js client

Teacher code: https://github.com/StephenGrider/ticketing

### Steps:

- create a Next.js client docker image. No TS, since more of a hassle.
- declare the new docker image to the skaffold artifacts to watch
- create its k8s deployment and service yaml file
- create the routing rule for ingress-nginx:

```
- path: /?(*)
  backend:
    serviceName: client-srv
    servicePort: 3000
```

### Server side fetching of data

```js
// pages/index.js

import axios from 'axios';

const LandingPage = ({ currentUser }) => {
  return <h1>Landing page</h1>;
};

/**
 * `getInitialProps` is a special Next.js method:
 *   - executed on the server
 *   - before the render
 *   - useful to pass data to the component
 */
LandingPage.getInitialProps = async () => {
  const response = await axios.post('/api/users/currentuser');

  return response.data;
};

export default LandingPage;
```

### Solving a curious network problem

The POST request to `/api/users/currentuser` is executed on the server, during `getInitialProps`, inside a kubernetes container.

The request will be handled by the node.js HTTP client, and use this full URL: `127.0.0.1:80/api/users/currentuser`.

However this URL will not be received by the ingress Nginx, so it will not be routed to the `auth` service.

**2 possible solutions:**

1. redirect the request to the ingress
2. redirect the request directly to the auth service => bad idea to put this knowledge inside the React code, which will have to know the relation _route => service_.

So the first option is the best one. Ingress Nginx already has the knowledge of the mapping _route => service_.

**But what domain should we use to reach to the ingress Nginx while we are inside a pod?**

**Second challenge:** we need to set the cookie on the server-side by extracting it from the client request.

The containers and the Ingress Nginx are in a different namespace. The containers are inside the `default` namespace. And the Ingress Nginx is inside the `ingress-nginx` namespace. This can be checked with `kubectl get namespace`;

So to communicate across namespace, the URL will be:

```
http://NAME_OF_SERVICE.NAMESPACE.svc.cluster.local
```

To get the service name, you can use: `kubectl get services`. However this will only list the services from the `default` namespace.

So we'll have to use:

```
kubectl get services -n ingress-nginx
```

Depending in the platform, the LoadBalancer service name might be different on your machine (ex: _ingress-nginx-controller_).

This is the final URL to use to get to the Ingress Nginx service:

```
http://ingress-nginx.ingress-nginx.svc.cluster.local
```

**Optional, but useful**: This long URL is annoying to write, so we can make it shorter, and use **external name service**.

### Applying this solution in the code

One BIG thing to be aware of: `getInitialProps` is not always executed on the server, as previously stated. Instead, it is is executed on the client in one specific case: _when navigating from one page to another while in the app (client-side navigation)._

**Second problem**: the server-side request has no `host`, and the routing rule inside _ingress-nginx_ will not recognize it.
SOLUTION: add the `Host` header to the request.

**Third problem**: cookie is missing on the server-side.
SOLUTION: Use the argument `req` inside `getInitialProps`. More specifically: `req.headers.Cookie`.

Finally this is how the request will look like:

```js
LandingPage.getInitialProps = async ({ req }) => {
  if (typeof window === `undefined`) {
    // we are on the server
    // Requests should be made using the cross-namespace URL
    const { data } = await axios.get(
      'http://ingress-nginx.ingress-nginx.svc.cluster.local/api/users/currentuser',
      {
        headers: {
          Host: 'ticketing.dev',
          Cookie: req.headers.cookie,
        },
      }
    );
    return data;
  } else {
    // we are on the browser.
    // Requests should be made to the relative path
    const { data } = await axios.get('/api/users/currentuser');
    return data;
  }
};
```

The `req` actually also includes the `Host` header. So we can simply forward ALL the headers:

```js
const { data } = await axios.get('...', {
  headers: req.headers,
});
```

### Reusable API client

The code above is really headache to write, and handle the client and server difference.

`api/buildClient.js`

```js
import axios from 'axios';

export default ({ req }) => {
  if (typeof window === 'undefined') {
    return axios.create({
      baseURL: 'http://ingress-nginx.ingress-nginx.svc.cluster.local',
      headers: req.headers,
    });
  } else {
    return axios.create({
      baseURL: '/',
    });
  }
};
```

To use it:

```js
LandingPage.getInitialProps = async (context) => {
  const client = buildClient(context);
  const { data } = await client.get('/api/users/currentUser');
  return data;
};
```

### Signout

Call the signout endpoint from the user browser. Not from the server-side, since we need to _propagate_ the empty cookie to the browser.
