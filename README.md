# Docker & Jenkins Automated Deployment Environment

This repository provides instructions for setting up an automated deployment environment using Docker Compose and Jenkins for the following components:

- [Playground-mvc](https://github.com/marlgrgr/PlaygroundMVC)
- [Movie-Review](https://github.com/marlgrgr/MovieReview)

This guide explains how to deploy all dependencies in Docker containers and use Jenkins to deploy the latest repository changes to a local environment container, maintaining centralized and secure configuration using Jenkins tools.


## Setting Up Docker Compose

**1.** First, customize the provided `docker-compose.yml` to suit your machine's needs. You can choose which services to deploy based on your requirements.

   > **Note**: For Jenkins to use the host machine's Docker, it needs access to port 2375. If you have Docker Desktop, enable this in Settings->General->Expose daemon on tcp://localhost:2375 without TLS. If not, use the included 'socat' container.

**2.** If you already have services like PostgreSQL, MongoDB, or ActiveMQ running on your host machine, you can choose not to include these in your Docker setup.

**3.** Build the Jenkins image with Docker installed:

```bash
docker-compose build jenkins
```

**4.** Start Jenkins and other required services. To start all services:

```bash
docker-compose up -d
```

   Or to start only specific services (e.g., Jenkins, Redis, and ActiveMQ):

```bash
docker-compose up -d redis activemq jenkins
```

   Available services in the docker-compose: mongo, postgres, redis, activemq, jenkins, localstack, socat

## Installing and Configuring Jenkins

**1.** Access the Jenkins web interface:

```
http://localhost:8080
```

**2.** Retrieve the initial admin password:

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

**3.** Enter the password in the web console.

**4.** Select "Install suggested plugins" when prompted.

**5.** Create your custom admin user and complete the setup.

**6.** Keep the default Jenkins URL (http://localhost:8080) and save.

## Configuring Jenkins Plugins and Tools

**1.** Install necessary plugins:
   - Go to "Manage Jenkins" > "Plugins"
   - In "Available plugins", search and install:
     - Eclipse Temurin installer (for JDK configuration)
     - Config File Provider (for configuration files)
     - HTML Publisher (for Jacoco reports)
     - NodeJS (for Node.js configuration)

**2.** Configure required tools:
   - Go to "Manage Jenkins" > "Tools"
   
   - For JDK:
     - Click "Add JDK"
     - Name: `JDK 21`
     - Check "Install automatically"
     - Click "Add installer" > "Install from adoptium.net"
     - Select the latest JDK 21 version
   
   - For Maven:
     - Name: `Maven 3`
     - Ensure the latest Maven 3 version is selected
   
   - For NodeJS:
     - Name: `Node-24`
     - Check "Install automatically"
     - Select the latest Node 24 version
   
   - Click "Save"

## Setting Up Configuration Files in Jenkins

**1.** **Backend Application Properties**:
   - Go to "Manage Jenkins" > "Managed Files"
   - Add new config > Select "Custom file"
   - ID: `playground-application-properties-local`
   - Give the file a name
   - Add content like:

```properties
spring.application.name=PlaygroundMVC
server.port=8081
spring.datasource.url=databaseEndpoint
spring.datasource.username=username
spring.datasource.password=password
spring.datasource.driver-class-name=org.postgresql.Driver
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.ddl-auto=update
spring.data.mongodb.uri=mongoUrl
management.endpoints.web.exposure.include=health,info,prometheus,metrics
management.endpoint.prometheus.access=read-only
jwt.secretKey=secretKey
jwt.expirationTime=3600
admin.default.user=admin
admin.default.password=admin123
redis.config.path=redisConfigPath
redis.config.maxIdleTime=3
redis.config.redisTTL=0
sqs.queue.region=region
sqs.queue.accessKey=accessKey
sqs.queue.secretKey=secretKey
spring.cloud.aws.sqs.enabled=false
sqs.queue.loadMovies.url=url
sqs.queue.loadMovies.name=name
api.movie.baseUrl=https://api.themoviedb.org/
api.movie.discover.path=3/discover/movie?include_adult=false&include_video=false&language=en-US&page=1&region=US&sort_by=popularity.desc&with_release_type=3
api.movie.genre.path=3/genre/movie/list?language=en
api.movie.apiKey=apiKey
spring.activemq.broker-url=activemqUrl
cors.allowed-origins=allowed,origins
```

2. **Redis Configuration File**:
   - Go to "Manage Jenkins" > "Managed Files"
   - Add new config > Select "Custom file"
   - ID: `playground-redisson-local`
   - Give the file a name
   - Add content:

```yaml
singleServerConfig:
 address: "redis://host.docker.internal:6379"
```

3. **Frontend Environment File**:
   - Go to "Manage Jenkins" > "Managed Files"
   - Add new config > Select "Custom file"
   - ID: `movie-review-properties-local`
   - Give the file a name
   - Add content:

```properties
VITE_API_BASE_URL=http://localhost:8081/api/v1
VITE_API_BASE_URL_WS=ws://localhost:8081/ws
VITE_DEBUG_MODE=false
```

## Configuring Local SQS with LocalStack

If you want to use SQS without an AWS account, you can set it up with LocalStack:

**1.** Access the LocalStack container:

```bash
docker exec -u 0 -it localstack bash
```

**2.** Create the SQS queue:

```bash
awslocal sqs create-queue --queue-name movie-load-queue
```

**3.** Verify the queue was created:

```bash
awslocal sqs list-queues
```

**4.** Exit the container:

```bash
exit
```

## Setting Up Jenkins Credentials

**1.** Configure secure credentials for URLs, usernames, passwords, and API keys:
   - Go to "Manage Jenkins" > "Credentials"
   - Select the global domain
   - Click "Add Credentials"
   - Choose "Secret text"
   - ID: `playground-credentials-local`
   - In the Secret field, add parameters that should replace placeholders in the application.properties file.

   Example for Docker container setup:

```
-Dspring.datasource.url=jdbc:postgresql://host.docker.internal:5432/appdb -Dspring.datasource.username=admin -Dspring.datasource.password=admin -Dspring.data.mongodb.uri=mongodb://host.docker.internal:27017/movieDb -Dspring.activemq.broker-url=tcp://host.docker.internal:61616 -Dsqs.queue.region=us-east-1 -Dsqs.queue.accessKey=test -Dsqs.queue.secretKey=test -Dspring.cloud.aws.sqs.enabled=true -Dsqs.queue.loadMovies.url=http://sqs.us-east-1.localhost.localstack.cloud:4566/000000000000/movie-load-queue -Dsqs.queue.loadMovies.name=movie-load-queue -Dredis.config.path=/app/config/redisson.yaml -Dapi.movie.apiKey=<<HERE_THE_API_KEY_FOR_EXTERNAL_MOVIE_ENDPOINT>> -Djwt.secretKey=<<JWT_SECRET_KEY>> -Dcors.allowed-origins=http://localhost:5173,http://localhost:3000,http://localhost:4173/ -Dsqs.queue.endpointOverride=http://host.docker.internal:4566
```

   **Important Notes**:
   
   - Default URLs/ports/users/passwords can be changed as needed
   - For `api.movie.apiKey`, you can generate a token as explained in [The Movie DB](https://developer.themoviedb.org/reference/intro/getting-started)
   - For SQS with LocalStack, you need to set up property `sqs.queue.endpointOverride`
   - Configure your `jwt.secretKey` here
   - The `cors.allowed-origins` contains typical ports used by Movie-Review in development, preview and production modes

## Creating Jenkins Pipelines

### Backend Pipeline (Playground-MVC)

1. From the Jenkins dashboard, click "New Item"
2. Enter a name for the pipeline
3. Select "Pipeline" and click "OK"
4. Under "Pipeline definition", select "Pipeline script from SCM"
5. In SCM, select "Git"
6. Repository URL: `https://github.com/marlgrgr/PlaygroundMVC.git`
7. Branch: `*/main`
8. Click "Save"
9. Run the pipeline with "Build now"

> **Note**: After the first execution, the action will change to "Build with parameters". You'll need to specify whether to run the pipeline for local or production. For production, you'll need to create additional configuration files (`playground-application-properties-production`, `playground-redisson-production`) and credentials (`playground-credentials-production`).

### Frontend Pipeline (Movie-Review)

1. From the Jenkins dashboard, click "New Item"
2. Enter a name for the pipeline
3. Select "Pipeline" and click "OK"
4. Under "Pipeline definition", select "Pipeline script from SCM"
5. In SCM, select "Git"
6. Repository URL: `https://github.com/marlgrgr/MovieReview.git`
7. Branch: `*/main`
8. Click "Save"
9. Run the pipeline with "Build now"

> **Note**: Similar to the backend pipeline, after the first execution, you'll be able to choose between local and production environments. For production, create an additional configuration file: `movie-review-properties-production`.
