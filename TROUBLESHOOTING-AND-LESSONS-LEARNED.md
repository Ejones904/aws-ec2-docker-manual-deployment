# Troubleshooting and Lessons Learned

## AWS EC2 Docker Manual Deployment

This document captures the primary issues I encountered while manually deploying a containerized Spring Boot application to AWS EC2, how I investigated them, what the root causes were, and what I learned from each incident.

The deployment ultimately succeeded with the application publicly accessible through the EC2 instance and returning an HTTP `200 OK`.

---

## Incident 1 — Docker Access Required Elevated Permissions

### Problem

After installing and starting Docker on the EC2 instance, Docker needed to be usable by the standard `ec2-user` without requiring `sudo` for every command.

### Investigation

Docker itself was running successfully. I confirmed the daemon was active by inspecting the running processes.

The remaining issue was user authorization rather than the Docker service.

### Resolution

I added the current user to the Docker group:

```bash
sudo usermod -aG docker $USER
```

Because Linux group membership is established when a session begins, I disconnected from the EC2 instance and established a new SSH session.

I then verified Docker access with:

```bash
docker ps
```

### Lesson Learned

Installing and starting a service does not automatically mean the current Linux user has permission to interact with it.

This reinforced the distinction between:

```text
Service availability
        vs.
User authorization
```

It also reinforced that some Linux permission changes require a new login session before they take effect.

---

# Incident 2 — Application Reachable but Returning HTTP 404

## Problem

After successfully pulling the application image from Docker Hub and starting the container with:

```bash
docker run -d -p 3000:8080 <image>
```

I accessed the application through the EC2 public IP on port `3000`.

Instead of receiving the expected response, Spring Boot returned:

```text
Whitelabel Error Page

status=404
Not Found
```

## Investigation

My first step was determining how far the request had traveled.

Receiving a Spring Boot Whitelabel page was important evidence.

The request had successfully passed through:

```text
Browser
   ↓
AWS Security Group
   ↓
EC2 Network Interface
   ↓
Host Port 3000
   ↓
Docker Port Mapping
   ↓
Container Port 8080
   ↓
Tomcat
   ↓
Spring Boot
```

This meant changing AWS networking or Docker port mappings would not address the actual problem.

I inspected the container logs:

```bash
docker logs <container>
```

The logs showed:

```text
Tomcat initialized with port 8080
Java app started
Tomcat started on port 8080
Started Application
```

This further confirmed that the application runtime itself had started successfully.

I then inspected the Java source code.

The application contained:

```java
public String getStatus() {
    return "OK";
}
```

However, the method was not exposed as an HTTP endpoint.

## Root Cause

The application had a method capable of returning the expected response, but Spring Boot had no controller mapping telling it to execute that method when `/` was requested.

## Resolution

I added REST controller functionality:

```java
@RestController
```

and mapped the root endpoint:

```java
@GetMapping("/")
public String getStatus() {
    return "OK";
}
```

The application then needed to be rebuilt and redeployed.

## Lesson Learned

An HTTP `404` does not automatically indicate a networking problem.

In this case, the `404` was actually useful evidence that the networking stack was functioning.

The important troubleshooting question became:

> How far did the request successfully travel?

Instead of changing several components at once, I could eliminate entire layers based on the response I received.

---

# Incident 3 — Source Code Changed but Deployment Did Not

## Problem

After correcting the Spring Boot endpoint, the newly deployed application still did not behave as expected.

The source code contained the correct annotations:

```java
@RestController
```

and:

```java
@GetMapping("/")
```

but the deployed application continued behaving like the previous version.

## Investigation

I verified the source code directly:

```bash
grep -n "RestController\|GetMapping" \
src/main/java/com/example/Application.java
```

The expected annotations were present.

That meant there was a difference between:

```text
Current Source Code
```

and:

```text
Artifact Packaged Into Docker Image
```

The Docker image was packaging the Maven-generated JAR.

Changing Java source code did not automatically mean the previously generated JAR contained those changes.

## Root Cause

The Docker image had been built using an application artifact that did not contain the latest source changes.

## Resolution

I rebuilt the Maven artifact:

```bash
mvn clean package
```

I then rebuilt the Docker image without using cached layers:

```bash
docker build --no-cache -t <image>:<new-tag> .
```

The new image was pushed to Docker Hub and pulled onto the EC2 deployment host.

## Lesson Learned

A successful source-code change is not the same thing as a successful deployment.

There are multiple artifacts in the deployment chain:

```text
Source Code
    ↓
Compiled Application
    ↓
JAR
    ↓
Docker Image
    ↓
Container Registry
    ↓
EC2
    ↓
Running Container
```

If one stage contains an older artifact, the deployed application can differ from the source code currently visible in Git.

This reinforced the importance of understanding exactly what artifact a Docker build is packaging.

---

# Incident 4 — ClassNotFoundException After Rebuild

## Problem

After rebuilding and deploying the updated application, the container was created successfully:

```bash
docker run -d ...
```

but requests to the application failed:

```text
curl: (7) Failed to connect to localhost port 3000
```

Rather than assuming another network problem, I checked the container logs.

```bash
docker logs <container>
```

The logs showed:

```text
java.lang.ClassNotFoundException: com.example.Application
```

## Investigation

The Docker container itself could be created, but the Java application inside it was failing during startup.

The expected application class was:

```text
com.example.Application
```

The source file was located at:

```text
src/main/java/com/example/Application.java
```

I checked the Java package declaration and found that the source structure and expected package needed to match.

## Root Cause

The Java package/class structure did not match the application's expected main class.

Spring Boot attempted to load:

```text
com.example.Application
```

but the packaged application did not contain the class at that expected location.

## Resolution

I corrected the Java package declaration:

```java
package com.example;
```

I then rebuilt the Maven application:

```bash
mvn clean package
```

Before rebuilding Docker again, I verified the contents of the generated JAR:

```bash
jar tf target/*.jar | grep Application.class
```

The expected result was:

```text
BOOT-INF/classes/com/example/Application.class
```

Only after verifying the artifact did I rebuild and push another Docker image.

## Lesson Learned

A running Docker container and a running application are not the same thing.

`docker run` returning a container ID only confirms that Docker accepted the request to create and start the container.

The application inside that container can still immediately fail.

Commands such as:

```bash
docker ps -a
docker logs <container>
```

are therefore essential when a newly started container does not respond.

This incident also reinforced the importance of verifying build artifacts instead of assuming a successful build produced the expected contents.

---

# Incident 5 — Verifying the Deployment Layer by Layer

After correcting the application package, rebuilding the JAR, rebuilding the Docker image, pushing the new version to Docker Hub, and redeploying it to EC2, I tested the application from the EC2 host first.

```bash
curl http://localhost:3000/
```

Result:

```text
OK
```

I then requested the HTTP headers:

```bash
curl -i http://localhost:3000/
```

Result:

```text
HTTP/1.1 200
Content-Type: text/plain;charset=UTF-8
Content-Length: 2
```

Finally, I accessed the application through the EC2 public endpoint from a browser.

The browser returned:

```text
OK
```

This provided end-to-end confirmation that the complete deployment path was functioning.

---

# Key Lessons Learned

## 1. Troubleshoot by Layer

One of the biggest lessons from this deployment was to determine which layer is failing before making changes.

The environment contained several distinct layers:

```text
Client
  ↓
AWS Networking
  ↓
Security Group
  ↓
EC2 Host
  ↓
Docker Engine
  ↓
Port Mapping
  ↓
Container
  ↓
Java Runtime
  ↓
Tomcat
  ↓
Spring Boot
  ↓
Application Route
```

Different symptoms provide evidence about how far a request traveled.

For example:

```text
Connection refused
```

and:

```text
HTTP 404 from Spring Boot
```

represent very different failures.

The 404 proved that the request reached the application.

---

## 2. Logs Are More Valuable Than Guessing

When the new container failed, the fastest path to the root cause was:

```bash
docker logs <container>
```

That immediately exposed:

```text
ClassNotFoundException
```

Without checking the logs, it would have been easy to unnecessarily change ports, Security Groups, or Docker configuration.

---

## 3. Understand the Artifact Chain

A deployment does not go directly from source code to production.

In this project:

```text
Java Source
   ↓
Maven
   ↓
Executable JAR
   ↓
Docker Build
   ↓
Docker Image
   ↓
Docker Hub
   ↓
docker pull
   ↓
EC2 Container
```

Each stage can contain a different version of the application if artifacts are not rebuilt and versioned correctly.

---

## 4. Versioned Images Make Troubleshooting Easier

Using explicit Docker image tags made it possible to distinguish one deployment attempt from another.

Instead of repeatedly overwriting:

```text
latest
```

new versions could be built, pushed, pulled, and tested independently.

This creates a clearer deployment history and reduces ambiguity about which artifact is actually running.

---

## 5. Manual Deployment Makes CI/CD Easier to Understand

Working through the deployment manually clarified what a deployment pipeline eventually needs to automate.

A CI/CD system would need to perform many of the same actions:

```text
Build application
        ↓
Package artifact
        ↓
Build container
        ↓
Tag image
        ↓
Authenticate with registry
        ↓
Push image
        ↓
Select deployment target
        ↓
Pull correct version
        ↓
Replace running workload
        ↓
Validate application health
```

The automation does not eliminate these dependencies.

It executes them consistently.

Understanding the manual process makes it easier to design the automation and troubleshoot it when something fails.

---

# Final Outcome

The deployment ultimately produced a publicly accessible Spring Boot application running inside Docker on AWS EC2.

Final validation:

```text
HTTP/1.1 200

OK
```

More importantly, the deployment required troubleshooting across application routing, Maven artifacts, Docker images, Java package structure, container logs, port mapping, and AWS networking.

The project provided practical experience not only deploying an application, but determining **where a deployment was failing and why**.

