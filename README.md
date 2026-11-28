# DevOps – Application Gestion des Projets

## 1. Project Overview

This project implements a Continuous Delivery (CD) pipeline for a Spring Boot application using Jenkins, Maven, Docker, Docker Hub, and MySQL.

The objective is to automate the application lifecycle, from retrieving the source code from GitHub to building the application, creating a Docker image, publishing it to Docker Hub, and deploying it into Docker containers.

The pipeline reduces manual deployment operations and ensures that the application can be built and deployed through a repeatable process.

---

## 2. Technologies Used

| Technology | Role |
|---|---|
| Java 21 | Development environment |
| Spring Boot | Backend application framework |
| Maven | Build and dependency management |
| Jenkins | CI/CD pipeline automation |
| Docker | Containerization |
| Docker Hub | Docker image registry |
| MySQL 8.0 | Relational database |
| Git & GitHub | Source code management |

---

## 3. Project Architecture

The application uses a containerized architecture composed of a Spring Boot backend and a MySQL database.

The backend container communicates with the MySQL container through a dedicated Docker network.

```text
                 Developer
                     |
                     v
                  GitHub
                     |
                     v
                  Jenkins
                     |
          +----------+----------+
          |                     |
          v                     v
     Maven Build           Docker Build
          |                     |
          +----------+----------+
                     |
                     v
              Docker Image
                     |
                     v
                 Docker Hub
                     |
                     v
               Deployment
                     |
          +----------+----------+
          |                     |
          v                     v
     Backend App             MySQL 8.0
     Port 8080               Port 3306
          |
          v
    REST API Endpoints
```

### Docker containers

| Container | Image | Purpose | Port |
|---|---|---|---|
| `backend-app` | `meriem0014/backend-app:latest` | Spring Boot REST API | 8080 |
| `mysql` | `mysql:8.0` | Database management | 3306 |

Both containers are connected to the `backend-network` Docker network.

---

## 4. Repository Structure

```text
DevOps-AppGestionDesProjets/
│
├── Jenkinsfile
│
└── backend/
    ├── Dockerfile
    ├── pom.xml
    ├── mvnw
    ├── src/
    │   ├── main/
    │   │   ├── java/
    │   │   └── resources/
    │   ││       └── application.properties
    │   │
    │   └── test/
    │
    └── target/
```

---

## 5. Continuous Delivery Pipeline

The Jenkins pipeline is defined in the `Jenkinsfile` at the root of the repository.

It automates the following stages:

### Stage 1 – Checkout

Jenkins retrieves the source code from the GitHub repository using the configured branch.

### Stage 2 – Build Maven

Maven compiles the Spring Boot application and packages it into a JAR file.

```bash
mvn clean package -DskipTests
```

The `-DskipTests` option skips the execution of tests during this pipeline build.

### Stage 3 – Docker Build

Docker builds an image containing the backend application using the Dockerfile.

The image is tagged locally as:

```text
backend-app:latest
meriem0014/backend-app:latest
```

### Stage 4 – Docker Push

Jenkins authenticates with Docker Hub using stored credentials and publishes the image to the Docker Hub repository.

```text
meriem0014/backend-app:latest
```

The credentials are managed through Jenkins Credentials and are not hardcoded into the pipeline.

### Stage 5 – Deploy MySQL

Jenkins ensures that the MySQL container is running.

The deployment uses:

- MySQL 8.0
- Database: `test_db`
- Docker network: `backend-network`
- Persistent volume: `mysql_data`
- Health check to verify database readiness

### Stage 6 – Deploy Backend

Jenkins deploys the backend application by pulling the Docker image from Docker Hub and launching the container.

The backend container:

- Uses the Docker network `backend-network`.
- Exposes port `8080`.
- Connects to MySQL using the hostname `mysql`.
- Uses environment variables for its database configuration.

### Stage 7 – Verify Deployment

Jenkins checks the deployed containers and displays the backend logs to verify that the application has started.

The pipeline reports its final execution status as successful or failed.

---

## 6. Docker Configuration

### Backend Dockerfile

The Dockerfile is located in the `backend/` directory.

It is responsible for creating the backend image, containing the Java runtime and the packaged Spring Boot application.

The resulting image is used by Jenkins for deployment.

### Docker Network

A dedicated Docker network is created:

```bash
docker network create backend-network
```

The backend connects to MySQL using the container name:

```text
mysql
```

This allows the application to reach the database internally without relying on `localhost`.

### Persistent Database Storage

A Docker volume is used to preserve MySQL data:

```text
mysql_data
```

This allows database data to persist beyond the lifecycle of the MySQL container.

---

## 7. Configuration

The application uses environment variables during deployment to configure the database connection.

| Variable | Description |
|---|---|
| `SPRING_DATASOURCE_URL` | MySQL connection URL |
| `SPRING_DATASOURCE_USERNAME` | Database username |
| `SPRING_DATASOURCE_PASSWORD` | Database password |

The deployed backend connects to MySQL using:

```text
jdbc:mysql://mysql:3306/test_db?createDatabaseIfNotExist=true
```

The database hostname `mysql` corresponds to the MySQL container name within the Docker network.

Sensitive credentials should be stored securely and never committed to GitHub.

---

## 8. Running the Application

### Prerequisites

- Java
- Maven
- Docker Engine
- Git

### Build the backend manually

```bash
cd backend
mvn clean package -DskipTests
```

### Build the Docker image

From the repository root:

```bash
docker build -t backend-app:latest ./backend
```

### Start MySQL

```bash
docker network create backend-network
```

```bash
docker volume create mysql_data
```

```bash
docker run -d \
  --name mysql \
  --network backend-network \
  -e MYSQL_ROOT_PASSWORD=root \
  -e MYSQL_DATABASE=test_db \
  -v mysql_data:/var/lib/mysql \
  -p 3306:3306 \
  mysql:8.0
```

### Start the backend

```bash
docker run -d \
  --name backend-app \
  --network backend-network \
  -p 8080:8080 \
  -e SPRING_DATASOURCE_URL="jdbc:mysql://mysql:3306/test_db?createDatabaseIfNotExist=true" \
  -e SPRING_DATASOURCE_USERNAME=root \
  -e SPRING_DATASOURCE_PASSWORD=root \
  backend-app:latest
```

---

## 9. Testing the Deployment

### Verify running containers

```bash
docker ps
```

### Verify Docker images

```bash
docker images
```

### Check backend logs

```bash
docker logs --tail 50 backend-app
```

### Test the REST API

```bash
curl -i http://localhost:8080/entreprise/all
```

Expected result:

```http
HTTP/1.1 200
Content-Type: application/json
```

The endpoint returns a JSON response containing the list of enterprises. If the database is empty, the response may be:

```json
[]
```

An HTTP 200 response confirms that the request reached the backend successfully and that the endpoint responded correctly.

---

## 10. Jenkins Configuration

The Jenkins pipeline requires Docker Hub credentials to publish the application image.

### Required Jenkins credential

| Field | Value |
|---|---|
| Credential type | Username with password |
| Username | Docker Hub username |
| Password | Docker Hub access token |
| Credential ID | `dockerhub-credentials` |

The credential ID must match the one referenced in the Jenkinsfile.

### Pipeline execution

The pipeline can be triggered through Jenkins by running the configured pipeline job.

The Jenkins Stage View provides a visual overview of the execution of each stage, while the Console Output provides detailed execution logs.

---

## 11. Results

The implemented pipeline automates the following operations:

- Retrieval of the application source code.
- Maven compilation and packaging.
- Docker image creation.
- Publishing the image to Docker Hub.
- Deployment of the MySQL database container.
- Deployment of the Spring Boot backend container.
- Verification of the deployed application.

The deployed backend was verified through Docker container status, application logs, and an HTTP request to the REST API.

---

## 12. Limitations and Future Improvements

Several improvements can be introduced to make the pipeline more robust and closer to production standards:

- Integrate automated unit and integration tests into the pipeline.
- Use a dedicated non-root MySQL user instead of the root account.
- Replace `latest` with versioned Docker image tags.
- Improve health checks for the backend application.
- Add automated rollback in case of deployment failure.
- Implement monitoring and alerting.
- Secure sensitive information using dedicated secrets management.
- Add SonarQube for static code analysis.
- Add automated testing before deployment.

---

## 13. Conclusion

This project demonstrates the implementation of a Continuous Delivery workflow using Jenkins, Maven, Docker, Docker Hub, Spring Boot, and MySQL.

The pipeline automates the main deployment steps, from source code retrieval to container deployment, making the application deployment process more reliable, reproducible, and easier to maintain.

It also provides a foundation for future DevOps improvements, including continuous integration, automated testing, security analysis, monitoring, and deployment optimization.
