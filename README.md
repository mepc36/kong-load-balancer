# Summary:

This repo is a minimal example for a kong load balancer. It is the result of following the tutorial in the Kong docs [here](https://docs.konghq.com/gateway/latest/get-started/load-balancing/).

## Starting Kong:

1. Clone the Docker repo:

```
git clone https://github.com/Kong/docker-kong
```

2. Change to the compose directory:

```
cd docker-kong/compose/
```

3. Start the Gateway stack:

```
KONG_DATABASE=postgres docker-compose --profile database up
```

4. Use the Admin API to create an upstream named `example_upstream`:

```
curl -X POST http://localhost:8001/upstreams \
  --data name=example_upstream
```

5. Create two targets for `example_upstream`, one with an `httpbun` connection endpoint and one with an `httpbin` connection endpoint:

```
curl -X POST http://localhost:8001/upstreams/example_upstream/targets \
  --data target='httpbun.com:80' && \
curl -X POST http://localhost:8001/upstreams/example_upstream/targets \
  --data target='httpbin.org:80'
```

6. Add new service:

```
curl -i -s -X POST http://localhost:8001/services \
  --data name=example_service \
  --data url='http://httpbin.org' \
  --data host='example_upstream'
```
7. Add `/mock` route to `example_service`:

```
curl -i -X POST http://localhost:8001/services/example_service/routes \
  --data 'paths[]=/mock' \
  --data name=example_route
```

8. Validate the loading balancing by pinging http://localhost:8000/mock/headers several times:

```
curl -s http://localhost:8000/mock/headers | grep -i -A1 '"host"'
```

The `host` value should change between `httpbin` and `httpbun`.
