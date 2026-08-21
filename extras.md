# Useful General CLI commands I might need

A markdown documentation of very useful cli commands:

## 1. Check your list of globally installed dependencies

npm list -g --depth=0

## 2. Swagger cli command to generate your documentation

swagger-cli bundle ./openapi.yaml --outfile ./dist/openapi.bundle.yaml --type yaml

## 2. docker cli command to run prometheus and mount the volume without using docker compose

docker run -p 9090:9090 `

> > -v ${PWD}\Docker\prometheus.yml:/etc/prometheus/prometheus.yml `
> > prom/prometheus
