# Useful Go CLI commands I might need

A markdown documentation of very useful but rarely used commands

## 1. Run Integration Tests
go test -tags=integration ./service/operation -v

## 2. Run Asynqmon (Web UI for asynq using docker)
PS C:\Users\Efosa> docker run --rm `
>>   --name asynqmon `
>>   -p 8088:8080 `
>>   -e REDIS_ADDR=host.docker.internal:6379 `
>>   hibiken/asynqmon
Asynq Monitoring WebUI server is listening on port 8080


## 3. Go into th container kernel to observe its content
docker exec -it d79d043fd6f5 sh

## 4. Block Go from reading your integration file unless specufied via tags
//go:build integration
// +build integration


## 5. Run a specfic test file
go test -tags=integration -v -run ^TestCreateUser$ ./service/operation
