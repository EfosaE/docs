# Docker Production Command Reference

## Quick Navigation
1. [Images](#images)
2. [Containers](#containers)
3. [Logs & Debugging](#logs--debugging)
4. [Docker Compose](#docker-compose)
5. [Networks & Volumes](#networks--volumes)
6. [System Cleanup](#system-cleanup)
7. [Registry Operations](#registry-operations)

---

## Images
```bash
# List images
docker images
docker images -a  # include intermediate layers

# Pull image
docker pull node:20-slim

# Build image
docker build -t myapp:latest .
docker build -f Dockerfile.prod -t myapp:prod .
docker build --no-cache -t myapp:latest .  # fresh build

# Tag image
docker tag myapp:latest myapp:v1.0.0
docker tag myapp:latest registry.example.com/myapp:latest

# Remove image
docker rmi myapp:old
docker rmi -f myapp:latest  # force remove

# Inspect image
docker inspect myapp:latest
docker history myapp:latest  # show layers

# Save/Load (for transfer)
docker save myapp:latest > myapp.tar
docker load < myapp.tar
```

---

## Containers
```bash
# Run container
docker run -d --name myapp -p 3000:3000 myapp:latest
docker run -d --name myapp -p 3000:3000 -e NODE_ENV=production --restart=unless-stopped myapp

# List containers
docker ps  # running only
docker ps -a  # all containers

# Start/Stop/Restart
docker start myapp
docker stop myapp
docker restart myapp
docker stop $(docker ps -q)  # stop all running

# Remove container
docker rm myapp
docker rm -f myapp  # force remove running container
docker rm $(docker ps -aq)  # remove all stopped

# Execute command in running container
docker exec -it myapp bash
docker exec -it myapp sh  # for alpine
docker exec myapp ls -la /app

# Inspect container
docker inspect myapp
docker inspect --format='{{.State.Running}}' myapp
docker inspect --format='{{.NetworkSettings.IPAddress}}' myapp

# Copy files
docker cp myapp:/app/logs/error.log ./error.log
docker cp ./config.json myapp:/app/config/

# View resource usage
docker stats myapp
docker stats  # all containers
```

---

## Logs & Debugging
```bash
# View logs
docker logs myapp
docker logs -f myapp  # follow (real-time)
docker logs -f --tail 100 myapp  # last 100 lines + follow
docker logs --since 30m myapp  # last 30 minutes
docker logs --timestamps myapp

# Check container processes
docker top myapp

# View port mappings
docker port myapp

# Check filesystem changes
docker diff myapp
```

---

## Docker Compose
```bash
# Start services
docker-compose up -d
docker-compose up -d --build  # rebuild images first
docker-compose up -d --force-recreate  # force recreate containers

# Stop services
docker-compose stop
docker-compose down  # stop and remove containers/networks
docker-compose down -v  # also remove volumes

# View logs
docker-compose logs -f
docker-compose logs -f --tail=100 app
docker-compose logs -f app db  # specific services

# List services
docker-compose ps

# Execute commands
docker-compose exec app bash
docker-compose exec -u root app bash
docker-compose run --rm app npm test  # one-off command

# Restart services
docker-compose restart
docker-compose restart app  # specific service

# Build services
docker-compose build
docker-compose build --no-cache app

# Pull latest images
docker-compose pull

# View configuration
docker-compose config  # validate and view resolved config

# Scale services
docker-compose up -d --scale worker=3
```

---

## Networks & Volumes

### Networks
```bash
# List networks
docker network ls

# Create network
docker network create mynetwork
docker network create --subnet=172.20.0.0/16 mynetwork

# Inspect network
docker network inspect mynetwork

# Connect/disconnect container
docker network connect mynetwork myapp
docker network disconnect mynetwork myapp

# Remove network
docker network rm mynetwork
docker network prune  # remove unused
```

### Volumes
```bash
# List volumes
docker volume ls

# Create volume
docker volume create mydata

# Inspect volume
docker volume inspect mydata
docker volume inspect --format='{{.Mountpoint}}' mydata

# Remove volume
docker volume rm mydata
docker volume prune  # remove unused

# Use volume with container
docker run -v mydata:/app/data myapp
docker run -v /host/path:/container/path:ro myapp  # read-only bind mount
```

---

## System Cleanup
```bash
# Remove stopped containers
docker container prune

# Remove unused images
docker image prune  # dangling only
docker image prune -a  # all unused

# Remove unused volumes
docker volume prune

# Remove unused networks
docker network prune

# Clean everything (CAREFUL!)
docker system prune  # containers, networks, dangling images
docker system prune -a  # also unused images
docker system prune -a --volumes  # everything including volumes

# Check disk usage
docker system df
docker system df -v  # detailed
```

---

## Registry Operations
```bash
# Login
docker login
docker login registry.example.com

# Tag for registry
docker tag myapp:latest username/myapp:latest
docker tag myapp:latest registry.example.com/myapp:latest

# Push to registry
docker push username/myapp:latest
docker push registry.example.com/myapp:latest

# Pull from registry
docker pull username/myapp:latest
docker pull registry.example.com/myapp:latest

# Logout
docker logout
```

---

## Quick Reference Commands
```bash
# Health check
docker inspect --format='{{.State.Health.Status}}' myapp

# Get container IP
docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' myapp

# Stop all running containers
docker stop $(docker ps -q)

# Remove all containers
docker rm $(docker ps -aq)

# Remove all images
docker rmi $(docker images -q)

# Follow logs for multiple containers
docker logs -f container1 & docker logs -f container2

# System info
docker version
docker info

# Update container resources
docker update --memory="1g" --cpus="2" myapp

# Restart policy update
docker update --restart=always myapp
```

---

## Common Production Patterns

### Deploy new version
```bash
docker pull myapp:latest
docker stop myapp
docker rm myapp
docker run -d --name myapp -p 3000:3000 --restart=unless-stopped myapp:latest
```

### With Docker Compose
```bash
docker-compose pull
docker-compose up -d
```

### Backup volume
```bash
docker run --rm -v mydata:/data -v $(pwd):/backup alpine tar czf /backup/backup.tar.gz /data
```

### Restore volume
```bash
docker run --rm -v mydata:/data -v $(pwd):/backup alpine tar xzf /backup/backup.tar.gz -C /
```

### Hot reload logs to file
```bash
docker logs -f myapp > app.log 2>&1 &
```

### Check if container is healthy
```bash
docker inspect --format='{{.State.Health.Status}}' myapp
# Output: healthy, unhealthy, or starting
```


### Run Asynqmon (Web UI for asynq using docker)
```bash 
     docker run --rm `
>>   --name asynqmon `
>>   -p 8088:8080 `
>>   -e REDIS_ADDR=host.docker.internal:6379 `
>>   -d
>>   hibiken/asynqmon
```
Asynq Monitoring WebUI server is listening on port 8080


### 3. Go into th container kernel to observe its content
docker exec -it d79d043fd6f5 sh