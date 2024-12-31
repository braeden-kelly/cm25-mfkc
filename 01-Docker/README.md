# Running a microservices app with Docker

Nothing requires us to use an orchestration framework. We could use vanilla
Docker to deploy our application.

## Bookinfo

For example, let's consider the fairly simple application shown below, known as
**Bookinfo**. 

![Bookinfo](../bookinfo-basic.svg)
<img src="../bookinfo-basic.svg">

Requests come into `productpage`. That service makes requests to `reviews` and
`details` to gather the information to display.

get IP

```shell
#export DETAILS_HOSTNAME=localhost
#export DETAILS_SERVICE_PORT=9082
#export REVIEWS_HOSTNAME=localhost
#export REVIEWS_SERVICE_PORT=9081 should be -e
docker run --rm -d -p 9080:9080 docker.io/istio/examples-bookinfo-productpage-v1:1.20.2
docker run --rm -d -p 9081:9080 docker.io/istio/examples-bookinfo-reviews-v1:1.20.2
docker run --rm -d -p 9082:9080 docker.io/istio/examples-bookinfo-details-v1:1.20.2
```

http://localhost:9080/productpage

## Clean up

```shell
docker rm -f $(docker ps -q)
docker ps
```
