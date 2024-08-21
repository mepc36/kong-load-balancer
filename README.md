# Summary:

This repo is a minimal example. It shows how to load balance 2 separate LLM services behind a Kong API Gateway. It is a less trivial implementation of the tutorial found in the Kong docs [here](https://docs.konghq.com/gateway/latest/get-started/load-balancing/).

## Creating & Running the Load Balancer:

1. Run ollama on port 11434:

```
docker run --network compose_kong-net -d -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama
```

2. Run a second ollama service on host port 11435:

```
docker run --network compose_kong-net -d -v ollama:/root/.ollama -p 11435:11434 --name ollama_2 ollama/ollama

```

3. Start Kong inside a Docker container:

```
cd ./compose && KONG_DATABASE=postgres docker-compose --profile database up -d
```

4. Create an upstream named `ai_upstream`:

```
curl -X POST http://localhost:8001/upstreams --data name=ai_upstream
```

5. Create two targets for `ai_upstream` that point at the 2 ollama Docker containers you just created:

```
curl -X POST http://localhost:8001/upstreams/ai_upstream/targets --data target=host.docker.internal:11434
```

```
curl -X POST http://localhost:8001/upstreams/ai_upstream/targets --data target=host.docker.internal:11435
```

6. Create an `ai_load_balancing_service` service with `ai_upstream` as its host:

```
curl -i -s -X POST http://localhost:8001/services \
  --data name=ai_load_balancing_service \
  --data path=/api/chat \
  --data host='ai_upstream'
```

7. Add a route to the `ai_load_balancing_service`:

```
curl -i -X POST http://localhost:8001/services/ai_load_balancing_service/routes \
  --data 'paths[]=/api/chat' \
  --data name=ai_load_balancing_route
```

8. Follow your docker logs in order to track which of the 2 containers is being called:

```
docker logs -f CONTAINER_ID_HERE
```

9. Test your load balancer:

```
curl -X POST 'http://localhost:8000/api/chat' \
--header 'Content-Type: application/json' \
--data '{ "messages": [ { "role": "system", "content": "You are a mathematician" }, { "role": "user", "content": "What is 1+1?"} ], "model": "llama3.1", "stream": false }'
```

### Other Commands:

What follows is a list of commands that, while not strictly necessary, were helpful when developing this load balancer.

## Test Ollama Directly:

curl http://localhost:11434/api/chat -d '{
  "model": "llama3.1",
  "messages": [
    { "role": "user", "content": "why is the sky blue?" }
  ],
  "stream": false
}'

## Create & Test Trivial Kong Service:

1. Create service:

```
curl -i -s -X POST http://localhost:8001/services --data name=example_service --data url='http://httpbin.org'
```

2. Create route:

```
curl -i -X POST http://localhost:8001/services/example_service/routes --data 'paths[]=/mock' --data name=example_route
```

3. Test:

GET:

```
curl -X GET http://localhost:8000/mock/anything
```

POST:

```
curl -X POST http://localhost:8000/mock/anything --header 'Content-Type: application/json' --data '{ "messages": []}'
```

## Create an LLM Service (not using Kong's AI Proxy Plugin):

1. Create `ai_service` service:

```
curl -i -s -X POST http://localhost:8001/services \
  --data name=ai_service \
  --data url='http://host.docker.internal:11434/api/chat'
```

2. Create route:

```
curl -i -X POST http://localhost:8001/services/ai_service/routes \
  --data 'paths[]=/api/chat' \
  --data name=ai_route
```

3. Test:

```
curl --location 'http://localhost:8000/api/chat' \
--header 'Content-Type: application/json' \
--data '{ "messages": [ { "role": "system", "content": "You are a mathematician" }, { "role": "user", "content": "What is 1+1?"} ], "model": "llama3.1", "stream": false }'
```
