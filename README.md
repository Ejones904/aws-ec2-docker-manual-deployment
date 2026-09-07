# AWS EC2 Docker Manual Deployment

Manual deployment of a containerized **Spring Boot application to AWS EC2**, using Docker Hub as the image registry and AWS Security Groups for network access control.

The project was designed to establish and troubleshoot the complete deployment path manually before automating the same process through CI/CD.

---

## Project Overview

The objective was to deploy a versioned Docker image to an AWS-hosted Linux server and validate the application from end to end.

The implementation included:

* AWS EC2 provisioning
* Amazon Linux administration
* SSH key-based access
* Docker installation and configuration
* Docker Hub image retrieval
* versioned container deployment
* Docker port mapping
* AWS Security Group configuration
* Spring Boot application troubleshooting
* HTTP endpoint validation

The final application was successfully deployed and returned:

```text
HTTP/1.1 200
OK
```

---

## Architecture

```text
Application Source
       |
       v
   Maven Build
       |
       v
  Docker Image
       |
       v
   Docker Hub
       |
       | docker pull
       v
+---------------------------+
|        AWS EC2            |
|                           |
|     Amazon Linux          |
|          |                |
|          v                |
|     Docker Engine         |
|          |                |
|          v                |
|  Spring Boot Container    |
|       Port 8080            |
|          |                |
|          v                |
|   EC2 Host Port 3000      |
+------------|--------------+
             |
             v
      Security Group
        TCP 3000
             |
             v
        HTTP Client
```

---

## Technology Stack

| Technology          | Purpose                  |
| ------------------- | ------------------------ |
| AWS EC2             | Cloud compute            |
| Amazon Linux        | Deployment host          |
| AWS Security Groups | Network access control   |
| Docker              | Container runtime        |
| Docker Hub          | Container image registry |
| Spring Boot         | Application framework    |
| Java 17             | Application runtime      |
| Maven               | Build and packaging      |
| SSH                 | Remote administration    |
| curl                | HTTP validation          |
| Git / GitHub        | Source control           |

---

## Engineering Decisions

### Manual Deployment Before CI/CD

The deployment was intentionally completed manually before introducing Jenkins automation.

This exposed each layer involved in delivering the application:

```text
Build
  ↓
Artifact
  ↓
Container Image
  ↓
Registry
  ↓
Cloud Host
  ↓
Container Runtime
  ↓
Network Access
  ↓
Application
  ↓
Validation
```

Understanding these dependencies makes later CI/CD failures easier to isolate because the underlying deployment process is already known.

---

### Versioned Docker Images

The application was deployed using explicitly versioned Docker images rather than relying on `latest`.

Example:

```bash
docker pull ejones904/demo-app:1.2.5-42
```

Using version-specific tags makes it possible to identify exactly which application artifact is running on the server.

---

### Separate Host and Container Ports

The application listens on port `8080` inside the container while EC2 exposes the service through port `3000`.

```text
EC2 :3000
    ↓
Docker Port Mapping
    ↓
Container :8080
    ↓
Spring Boot
```

The container was deployed using:

```bash
docker run -d -p 3000:8080 ejones904/demo-app:1.2.5-42
```

---

### Layered Validation

Application availability was not treated as a single pass/fail condition.

The deployment was validated through multiple layers:

* EC2 instance state
* Docker daemon state
* container state
* application logs
* port mapping
* local HTTP response
* AWS Security Group
* external HTTP response

This approach became especially important during troubleshooting.

---

## Implementation

The EC2 instance was provisioned and accessed using SSH key authentication.

Docker was installed on the Amazon Linux host and the standard EC2 user was configured for Docker access.

A versioned Spring Boot image was pulled from Docker Hub and deployed with:

```bash
docker run -d \
  --name demo-app \
  -p 3000:8080 \
  ejones904/demo-app:<VERSION>
```

The AWS Security Group was then configured to permit the required inbound application traffic.

Detailed commands and chronological build steps are preserved in [`IMPLEMENTATION.md`](IMPLEMENTATION.md).

---

## Deployment Workflow

```text
Maven Application
       |
       v
Application JAR
       |
       v
Docker Image
       |
       v
Docker Hub
       |
       v
AWS EC2
       |
       v
Docker Container
       |
       v
Spring Boot :8080
       |
       v
EC2 :3000
       |
       v
HTTP Validation
```

### Container Deployment

The versioned image was successfully retrieved from Docker Hub.

![Docker Image Pulled](screenshots/06-docker-image-pulled-from-dockerhub.png)

The application container was then launched and its port mapping verified.

![Application Container Running](screenshots/07-application-container-running.png)

---

## Network Configuration

AWS Security Groups were used to control inbound traffic to the EC2 instance.

Application access was configured for TCP port `3000`, which maps to Spring Boot's internal container port `8080`.

![Security Group Port 3000](screenshots/08-security-group-port-3000.png)

SSH access and application access were treated as separate network requirements rather than broadly exposing the server.

---

## Troubleshooting

The most valuable part of this project was diagnosing failures across multiple layers of the deployment stack.

### HTTP 404 — Infrastructure Was Working

The first external request returned a Spring Boot **Whitelabel 404** page.

At first glance, the application appeared unavailable.

However, the 404 response provided useful evidence.

For Spring Boot to return the Whitelabel page, the request had already successfully traveled through:

```text
Internet
   ↓
AWS Security Group
   ↓
EC2 Port 3000
   ↓
Docker Port Mapping
   ↓
Container Port 8080
   ↓
Spring Boot
```

That allowed the infrastructure and networking layers to be ruled out.

The investigation moved to the application layer.

### Root Cause

The Java method existed:

```java
public String getStatus() {
    return "OK";
}
```

but it had not been exposed as an HTTP endpoint.

The application was updated with Spring REST mappings:

```java
@RestController
```

and:

```java
@GetMapping("/")
public String getStatus() {
    return "OK";
}
```

---

### Stale Application Artifact

After changing the Java source, rebuilding only the Docker image did not resolve the issue.

The investigation showed that Docker was still packaging the previously generated Maven artifact.

The application therefore had to be rebuilt first:

```bash
mvn clean package
```

followed by a new Docker image build.

This demonstrated an important dependency:

```text
Source Change
     ↓
Maven Artifact Rebuild
     ↓
Docker Image Rebuild
     ↓
Registry Push
     ↓
EC2 Redeployment
```

A source-code change does not automatically update an existing application artifact.

---

### ClassNotFoundException

A later container deployment failed to remain available.

Instead of immediately modifying AWS networking again, the container state and logs were inspected:

```bash
docker ps -a
docker logs demo-app
```

The logs showed:

```text
java.lang.ClassNotFoundException: com.example.Application
```

The failure was therefore inside the application artifact rather than EC2, the Security Group, or Docker networking.

### Root Cause

The Java package declaration did not correctly match the expected application package structure.

The package configuration was corrected and the resulting JAR was inspected before another Docker build:

```bash
jar tf target/*.jar | grep Application.class
```

The expected class was verified at:

```text
BOOT-INF/classes/com/example/Application.class
```

The Maven artifact and Docker image were then rebuilt and redeployed.

---

## Validation

### Spring Boot Startup

Container logs were inspected to confirm successful application initialization.

![Spring Boot Application Started](screenshots/09-spring-boot-application-started.png)

---

### Local EC2 Validation

The application was first tested directly from the EC2 host:

```bash
curl -i http://localhost:3000/
```

The resulting response was:

```text
HTTP/1.1 200
Content-Type: text/plain;charset=UTF-8
Content-Length: 2

OK
```

![Local HTTP 200 Validation](screenshots/10-local-http-200-validation.png)

Testing locally first separated application/container health from external AWS network access.

---

### External Validation

The application was then tested through the EC2 public endpoint.

![Public Browser Validation](screenshots/11-public-browser-validation.png)

The successful external request validated the complete path:

```text
Docker Hub
     ↓
AWS EC2
     ↓
Docker Engine
     ↓
Spring Boot :8080
     ↓
EC2 :3000
     ↓
Security Group
     ↓
Public Client
     ↓
HTTP 200 OK
```

---

## Security Considerations

Security controls implemented during the project included:

* SSH key-based EC2 authentication
* private SSH key excluded from GitHub
* AWS Security Group rules
* separation of SSH and application access
* non-root Docker administration
* controlled inbound access where practical

For a production implementation, additional controls would include:

* avoiding direct public exposure of the application container
* placing the application behind an ALB or reverse proxy
* HTTPS/TLS
* IAM-based administration where applicable
* centralized secret management
* automated patch management
* container vulnerability scanning
* centralized logging and monitoring
* least-privilege network rules

---

## What This Project Demonstrates

This project demonstrates practical experience with:

* AWS EC2
* AWS Security Groups
* Linux administration
* SSH authentication
* Docker installation
* Docker image management
* Docker Hub
* container deployment
* Docker port mapping
* Maven
* Spring Boot
* Java application troubleshooting
* container log analysis
* HTTP troubleshooting
* artifact lifecycle troubleshooting
* application vs. infrastructure fault isolation
* end-to-end deployment validation

---

## Future Enhancements

The manual workflow provides the foundation for automation.

Logical next steps include:

* automated deployment through Jenkins
* automated Docker image versioning
* application health checks
* deployment rollback
* Elastic IP or DNS-based addressing
* Application Load Balancer
* HTTPS
* centralized logging
* monitoring
* Terraform-based EC2 provisioning
* container registry and image security improvements

The deployment process established here is being extended into a larger **Jenkins multibranch CI/CD workflow**.

---

## Repository Documentation

* [`README.md`](README.md) — engineering overview, architecture, troubleshooting, and validation
* [`IMPLEMENTATION.md`](IMPLEMENTATION.md) — detailed chronological implementation record

---

## Engineering Outcome

The project established a complete manual deployment path from a versioned container image to a functioning AWS-hosted application.

More importantly, troubleshooting demonstrated how evidence from each layer can be used to narrow the scope of a failure.

A `404`, a stopped container, and a failed network connection may all appear to the user as an unavailable application, but they represent very different failure domains.

By validating each layer independently, the deployment was successfully taken from Docker Hub through AWS EC2 to a confirmed external **HTTP 200 OK** response.
