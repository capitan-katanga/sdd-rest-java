# Docker Guide for Spring Boot Applications

## Contents
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Development with Automatic Docker Compose Support](#development-with-automatic-docker-compose-support)
- [Available Docker Files](#available-docker-files)
- [Application Configuration](#application-configuration)
- [Best Practices](#best-practices)
- [Development vs Production](#development-vs-production)
- [Troubleshooting](#troubleshooting)
- [Quick Reference](#quick-reference)
- [Deployment Checklist](#deployment-checklist)
- [Resources](#resources)

## Overview
This guide covers Docker deployment for Spring Boot 4 applications using JVM-based images.

**Spring Boot 4 Requirements:**

1. Java 25 (LTS) — used in our Dockerfiles
2. Jakarta EE 11 / Servlet 6.1 baseline
3. PostgreSQL (we use `postgres:18-alpine` for optimal size and performance)

**Key Improvements in These Docker Files:**

1. Eclipse Temurin 25 official images (Alpine-based for smaller footprint)
2. PostgreSQL 18 Alpine (smaller, more secure)
3. Optimized JVM flags for container environments
4. curl installed for healthchecks
5. Non-root user security
6. Multi-stage builds for smaller images
7. ❌ **No Buildpacks/Jib** — stick to the provided Dockerfiles/Compose

## Prerequisites

1. Docker installed and running
2. Docker Compose (included with Docker Desktop)
3. Spring Boot application with Maven build configuration

## Development with Automatic Docker Compose Support

Spring Boot 4 includes the `spring-boot-docker-compose` dependency that automatically manages Docker containers during development. No more manual `docker compose up` commands!

### How It Works

When you run `./mvnw spring-boot:run`, Spring Boot will:
1. Detect the `compose.yaml` or `docker-compose.yml` file in your project root
2. Automatically start the PostgreSQL container defined in the compose file
3. Configure the datasource connection automatically
4. Stop the container when the application shuts down

### Setup

Create a `compose.yaml` file in your project root:

```yaml
services:
  postgres:
    image: postgres:18-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "user"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 512m
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```
> Compose spec v2+: omit the `version:` key. Spring Boot's `spring-boot-docker-compose` works with this layout.

### Usage

```bash
# Just run your application - PostgreSQL starts automatically!
./mvnw spring-boot:run
```

**No manual `docker compose up` needed!** Spring Boot handles container lifecycle automatically during development.

### Configuration

Add the dependency to your `pom.xml` (included by default in full-stack projects):

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-docker-compose</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>
</dependency>
```

You can customize behavior in `application.properties`:

```properties
# Disable Docker Compose support
spring.docker.compose.enabled=false

# Keep containers running after application stops (useful for debugging)
spring.docker.compose.lifecycle-management=start-only

# Specify custom compose file location
spring.docker.compose.file=docker/compose-dev.yaml
```

**Benefits:**

1. No manual container management during development
2. Automatic datasource configuration
3. Containers start only when needed
4. Automatic cleanup on application stop (configurable)
5. Works with PostgreSQL, MySQL, MongoDB, Redis, Kafka, and more

**Note:** This is for development only. For production deployment, see the sections below.

## Available Docker Files

### 1. Dockerfile (JVM-based)
Standard Docker deployment using Eclipse Temurin official Java images.

**Location**: Copy to your project root

**Features**:

1. Multi-stage build for smaller image size
2. Uses Eclipse Temurin 25 (official Java images)
3. Maven build included
4. Non-root user for security
5. Health check configured with curl
6. Optimized JVM flags for containers
7. Production-ready

**Build and Run**:
```bash
# Build the image
docker build -t my-spring-app .

# Run the container
docker run -p 8080:8080 my-spring-app
```

### 2. docker-compose.yml (JVM with Database)
Complete stack with PostgreSQL 18 database and Spring Boot application.

**Location**: Copy to your project root

**Features**:

1. PostgreSQL 18 Alpine (lightweight, production-ready)
2. Spring Boot application service
3. Automatic database connection configuration
4. Health checks for both services
5. Named volumes for data persistence
6. Network isolation
7. No version field (modern Docker Compose)

**Usage**:
```bash
# Start all services
docker compose up -d

# View logs
docker compose logs -f

# Stop all services
docker compose down

# Stop and remove volumes (deletes data)
docker compose down -v
```

**Access**:
- Application: http://localhost:8080
- Database: localhost:5432

## Application Configuration

### Environment Variables
Configure your application using environment variables in docker-compose.yml:

```yaml
environment:
  SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/mydb
  SPRING_DATASOURCE_USERNAME: user
  SPRING_DATASOURCE_PASSWORD: password
  SPRING_JPA_HIBERNATE_DDL_AUTO: update
  SERVER_PORT: 8080
```

### For Application-Only Deployment
If your application doesn't use a database, use the standalone Dockerfile:

```yaml
services:
  spring-app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: spring-boot-app
    ports:
      - "8080:8080"
    restart: unless-stopped
```

**Note**: Modern Docker Compose doesn't require a version field.

## Best Practices

### 1. Image Optimization
- Use multi-stage builds to minimize final image size
- Use Alpine or slim variants: `postgres:18-alpine`, `eclipse-temurin:25-jre-alpine`
- Clean up package manager cache after installations
- Copy only necessary files
- Use `.dockerignore` to exclude unnecessary files

### 2. Security
- Run as non-root user
- **Pin specific versions** in production (postgres:18-alpine, not latest)
- Use official images: Eclipse Temurin for Java, postgres:alpine for database
- Scan images for vulnerabilities: `docker scout cves my-app`
- Keep base images updated
- Avoid latest tag in production

### 3. Health Checks
- Always include health checks
- Use Spring Boot Actuator's health endpoint
- Configure appropriate intervals and timeouts

### 4. Resource Management
- Set memory limits in docker-compose.yml:
```yaml
    deploy:
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M
```

**JVM Container Optimization:**
The Dockerfile includes optimized JVM flags:
- `-XX:+UseContainerSupport`: Enables container-aware memory management
- `-XX:MaxRAMPercentage=75.0`: Uses 75% of container memory limit
- These flags ensure the JVM respects Docker memory limits

### 5. Data Persistence
- Use named volumes for database data
- Back up volumes regularly
- Consider using bind mounts for development

### 6. Networking
- Use custom networks for service isolation
- Expose only necessary ports
- Use service names for inter-container communication

## Development vs Production

### Development Setup
```bash
# Use docker-compose for easy setup
docker compose up

# Hot reload with spring-boot-devtools (mount source)
docker compose -f docker-compose-dev.yml up
```

### Production Setup
```bash
# Build production image
docker build -t myapp:1.0.0 .

# Tag and push to registry
docker tag myapp:1.0.0 myregistry.com/myapp:1.0.0
docker push myregistry.com/myapp:1.0.0

# Deploy with specific version
docker run -d -p 8080:8080 myregistry.com/myapp:1.0.0
```

## Troubleshooting

### Container Logs
```bash
# View application logs
docker logs spring-boot-app -f

# View all service logs
docker compose logs -f
```

### Database Connection Issues
```bash
# Check if PostgreSQL is ready
docker exec postgres-db pg_isready -U user

# Connect to database
docker exec -it postgres-db psql -U user -d mydb
```

### Performance Monitoring
```bash
# Check resource usage
docker stats

# View container details
docker inspect spring-boot-app
```

## Quick Reference

### Common Commands
```bash
# Build image
docker build -t my-app .

# Run container
docker run -p 8080:8080 my-app

# Start services
docker compose up -d

# Stop services
docker compose down

# View logs
docker compose logs -f spring-app

# Rebuild and restart
docker compose up -d --build

# Access container shell
docker exec -it spring-boot-app bash

# Clean up unused images
docker system prune -a
```

## Deployment Checklist
- [ ] Update database credentials (change from default user/password)
- [ ] Configure environment variables for production
- [ ] **Pin versions**: Use specific tags (postgres:18-alpine, not latest)
- [ ] **Java version**: Verify Java 25+ for Spring Boot 4
- [ ] Set up health checks (already configured in provided files)
- [ ] Configure resource limits (memory, CPU)
- [ ] Set up logging and log aggregation
- [ ] Configure backups for data volumes
- [ ] Test container restart behavior
- [ ] Document exposed ports
- [ ] Set up monitoring (Prometheus/Grafana)
- [ ] Configure reverse proxy (nginx/traefik) if needed
- [ ] Security scan: Run `docker scout cves` on built images
- [ ] Remove default credentials from docker-compose files

## Resources
- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Spring Boot Docker Guide](https://spring.io/guides/gs/spring-boot-docker/)
