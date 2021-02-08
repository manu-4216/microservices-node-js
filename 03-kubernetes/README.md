# Kubernetes

## Enabling it in Docker

_On Mac/Windows:_ Docker icon > Preferences > Kubernetes > Enable Kubernetes > Apply & Reset

_On Linux: install minikube_

Note: the docker driver is not compatible with ingress([issue](https://minikube.sigs.k8s.io/docs/drivers/docker/#known-issues)). To avoid this:

- option 1: pass a different `--driver`
- option 2: use `--vm=true`

```bash
// macOS:
minikube start --vm=true
minikube start --driver=hyperkit
minikube start --driver=virtualbox
```

Test the install:

```bash
kubectl version
```

## Creating a first config file: pod

See [pod.yaml](./pod.yaml)

Then run the pod with

```bash
kubectl apply -f pod.yaml
```

Once done with it, tear down the test pod with:

```bash
kubectl delete -f pod.yaml
```

## Creating a deployment

See [deployment.yaml](./deployment.yaml)

## Updating the code of a deployment

Rather than manually updating the image version in the yaml config file, we can use an image without a tag. This means that the latest image will be used. To redeploy, this command can be used:

```bash
kubectl rollout restart deployment <deploymentName>
```

## Services

Whenever we need communication across pods, we need a service.
When created, each Service is assigned a unique IP address (also called clusterIP). The service IP is completely virtual.

There are multiple types of services:

- **cluster IP** = setup an easy-to-remember URL to access a pod. Only exposes pods inside the cluster.
- **Node port** = makes a pod accessible from outside the cluster. Usually used for dev purposes.
- **load balancer** = makes a pod accessible from outside the cluster. This is the usual way to expose a pod to the outside world.
- **external name** = redirects an in-cluster request to a CNAME url.

So:

- for inter microservice communication: cluster IP
- for exposing the application to the outside world: _node port_, or a _load balancer_

### NodePort

Example of a NodePort: see [service-nodeport.yaml](./service-nodeport.yaml). This is only used for testing purposes.

This service will now be accessible via `localhost:<nodePort>`, where `nodePort` is the locally mapped port, that you can find listed with `kubectl get services`. It is a random port, like _30xxx_. This is not good for production, since the port is random, so we can't use it in the code.

### Cluster IP

This service is usually linked to a deployment (group of pods). So it's ok to colocate the configuration of the Cluster IP inside the deployment yaml.

To create multiple resources inside the same yaml file, we need to add `---`:

```yaml
<first resource definition>
---
<another resource definition>
```

See example in [deployment-with-cluster-ip.yaml](./03-kubernetes/deployment-with-cluster-ip.yaml).

## Communication between microservices

Whenever a microservice makes a request to another microservice (example: posts service sending an event to the event bus service), we will now _use the name of the service cluster IP_ in the URL.

Example:

```js
axios.post('http://event-bus-cluster-ip-srv/posts', {...});
```

## Connecting the services to the outside world

We will need 2 things:

1: **Load balancer Service**
Gets traffic to a single pod. Outside of the cluster, at the provider level (AWS, Azure, etc).

2: **Ingress** _aka_ **Ingress Controller** = pod with a set of routing rules, to distribute traffic to other services. This is inside the cluster.

This is exactly what we need to route the request path to different cluster IP services, corresponding to a microservice pod.

### Ingress-nginx

For the ingress, we will use `ingress-nginx` which is a third party solution.

Installation guide: https://kubernetes.github.io/ingress-nginx/deploy/

This will be created with a yaml config file. See it in [ingress-svc.yaml](./ingress-svc.yaml)

### Host file changes

Kubernetes is able to run multiple applications on different hosts.

The ingress has a property called `host` which points to a locally defined host. In order to make this work on the local machine, and recognise this as localhost, we need to edit the host file.

This is only needed for dev purposes.

Location on MacOS/Linux: `/etc/hosts`

### Deployment of the react app

This will need to be dockerized, then used inside a separate deployement + cluster IP.

### Adding routing rules inside the ingress-nginx

Reminder of the routes open to the outside world:

| method | path                | microservice | action                   |
| ------ | ------------------- | ------------ | ------------------------ |
| GET    | /                   | react client | get static assets        |
| GET    | /posts              | query        | get all posts + comments |
| POST   | /posts              | posts        | create a post            |
| POST   | /posts/:id/comments | comments     | create a comment         |

_PROBLEM: ingress-nginx cannot do routing based on the method (GET | POST)._
So we cannot differentiate between POST /posts and GET /posts.

To fix this, we first need to change the path, and make it unique (\*) across microservices:

| method | path                | microservice | action                   |
| ------ | ------------------- | ------------ | ------------------------ |
| GET    | /                   | react client | get static assets        |
| GET    | /posts              | query        | get all posts + comments |
| POST   | /posts/create (\*)  | posts        | create a post            |
| POST   | /posts/:id/comments | comments     | create a comment         |

The final routing rules are bellow:

```yaml
# ingress-svc.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress-srv
  annotations:
    kubernetes.io/ingress.class: nginx # needed for ingres-nginx
    nginx.ingress.kubernetes.io/use-regex: 'true' # allow use of regular expression in the paths
spec:
  rules:
    - host: posts.com # only useful for dev environment, for easy access. In the production application, this will be something like `tesla.posts
      http:
        paths:
          - path: /posts/create # posts.com/posts/create will point to posts cluster IP service
            backend:
              serviceName: posts-cluster-ip-srv
              servicePort: <clusterIPPort>
          - path: /posts
            backend:
              serviceName: query-cluster-ip-srv
              servicePort: <clusterIPPort>
          - path: /posts/?(.*)/comments # use regular expression nginx style. Needs 2nd annotation on top
            backend:
              serviceName: comments-cluster-ip-srv
              servicePort: <clusterIPPort>
          - path: /?(.*) # allow typical react SPA needs to use client-side routing. Put it at the end, to match last !
            backend:
              serviceName: client-cluster-ip-srv
              servicePort: <clusterIPPort>
```

This can also be found inside [ingress-svc-final.yaml](./ingress-svc-final.yaml).

## Avoiding the manual re-deploy when making changes

Current manual process:

- make a change in one/more services
- recreate the image, and push it
- run the command `kubectl rollout restart deployment <deplName>`

Solution:

_skaffold_ = "handles the workflow for building, pushing and deploying a k8s application"

Alternative: _helm_

Useful link about its purpose and alternative [here](https://medium.com/containers-101/the-ultimate-guide-for-local-development-on-kubernetes-draft-vs-skaffold-vs-garden-io-26a231c71210).

### Skaffold

Create a yaml configuration file. See [skaffold.yaml](./skaffold.yaml).

What will skaffold do automatically:

- watch the config files
- when we start skaffold: apply the config files ==> resources
- when we make a change: apply the config files ==> update resources
- when we stop skaffold: delete the obsolete resources

This tool is very useful when working with different projects.

#### Starting skaffold:

```bash
skaffold dev
```

**Note:** We use `nodemon` to propagate the changes inside the container, and restart the node service. Same for create-react-app. The changes will be picked up by hot reloading, which facilitates the dev process.

Stop skaffold: `^C`
