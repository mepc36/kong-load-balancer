# COMMANDS:

## TEST OLLAMA DIRECTLY:

curl http://localhost:11434/api/chat -d '{
  "model": "llama3.1",
  "messages": [
    { "role": "user", "content": "why is the sky blue?" }
  ],
  "stream": false
}'

## CREATE & TEST SIMPLE SERVICE:

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

## CREATE AN AI SERVICE NOT USING AI PROXY:

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

## LOAD BALANCE LLMS USING AN UPSTREAM & 2 TARGETS:

1. Run ollama on port 11434:

```
docker run --network compose_kong-net -d -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama
```

2. Run a second ollama service on port 11435:

```
docker run --network compose_kong-net -d -v ollama:/root/.ollama -p 11435:11435 --name ollama_2 ollama/ollama

```

3. Start Kong inside a Docker container:

```
cd ./compose && KONG_DATABASE=postgres docker-compose --profile database up -d
```

4. Create an `ai_service_2` service with an `ai_upstream` host:

```
curl -i -s -X POST http://localhost:8001/services \
  --data name=ai_service_2 \
  --data host='ai_upstream'
```

5. Create two targets for `ai_upstream`:

```
curl -X POST http://localhost:8001/upstreams/example_upstream/targets \
  --data target=host.docker.internal:11434' && \
curl -X POST http://localhost:8001/upstreams/example_upstream/targets \
  --data target=host.docker.internal:11435'
```

6. Add route:

```
curl -i -X POST http://localhost:8001/services/ai_service_2/routes \
  --data 'paths[]=/api/chat' \
  --data name=ai_load_balancing_route
```

7. Test ollama via Kong:

```
curl -X POST 'http://localhost:8000/api/chat' \
--header 'Content-Type: application/json' \
--data '{ "messages": [ { "role": "system", "content": "You are a mathematician" }, { "role": "user", "content": "What is 1+1?"} ], "model": "llama3.1", "stream": false }'
```
