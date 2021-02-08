# NATS streaming server

Next step: building the event-bus service.

Tackling this before other things (like building the orders service) is important for understanding some limitations of this pattern.

**Documentation**: docs.nats.io

NOTE: **NATS** is different than **NATS Streaming Server**
We will use **NATS Streaming Server** which is more advanced.

Docker image on docker hub: **nats-streaming**

## How to test the NATS service with Kubernetes

For easy testing of a failure, use Kubernetes port forwarding.
