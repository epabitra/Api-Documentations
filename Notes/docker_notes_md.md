# Docker Complete Notes

## Table of Contents

1. [Application Fundamentals](#application-fundamentals)
2. [Application Environments](#application-environments)
3. [Life without Docker](#life-without-docker)
4. [Life with Docker](#life-with-docker)
5. [Docker Architecture](#docker-architecture)
6. [Virtualization](#virtualization)
7. [Containerization](#containerization)
8. [Environment Setup](#environment-setup)
9. [Docker Commands](#docker-commands)
10. [Dockerfile](#dockerfile)
11. [Dockerizing Applications](#dockerizing-applications)
12. [Docker Network](#docker-network)
13. [Docker Compose](#docker-compose)
14. [Docker Volumes](#docker-volumes)
15. [Docker Swarm](#docker-swarm)
16. [Summary](#summary)

---

## Application Fundamentals

### What is an Application?

Application is nothing but a collection of programs.

### Application Tech Stack

Every Application contains 3 layers:

#### 1. Frontend (UI)
- Angular
- React JS
- Vue JS

#### 2. Backend (Business Logic)
- Java
- .Net
- Python
- Node JS
- PHP

#### 3. Database
- Oracle
- MySQL
- Postgres
- SQL Server
- Mongo DB

---

## Application Environments

In Realtime, our application will be deployed into Multiple Environments for Testing Purpose:

1. **DEV Env** - Developers Testing
2. **SIT Env** - Testing Team (QA) - System Integration
3. **UAT Env** - Client Side Testing - User Acceptance Testing
4. **PILOT Env (Pre-Production)** - Testing with Live Data

Once testing is completed in all above environments, it will be deployed into **PRODUCTION Env**.

- Production env means live environment
- End users will access application from Production env

---

## Life without Docker

- We need to install all the required software in all environments to run our application
- We need to make sure we are using the same versions of software in all machines
- If any software version is not matched, then application execution may fail

**Example:**
- Raja installed Java 11 v in Dev Env
- Sunil installed Java 8 v in SIT Env

If we want to run our application in multiple machines, then we have to install required software in all those machines, which is a hectic task.

---

## Life with Docker

- Docker is a containerization platform
- Docker is used to build and deploy our application into any machine without bothering about dependencies
- **Dependencies** means the software which are required to run our application
  - Dependencies = OS / Angular / React / Java / DB / Tomcat etc.
- Docker will reduce the gap between Development and Deployment

---

## Docker Architecture

1. **Dockerfile** - It contains instructions to build docker image
2. **Docker Image** - It is a package which contains code + dependencies
3. **Docker Registry** - It is a repository to store docker images
4. **Docker Container** - It is a runtime process which runs our application

**Note:** Once Docker image is created, then we can pull that image and we can run that image in any machine.

---

## Virtualization

- Installing Multiple Guest Operating Systems in one Host Operating System
- Hypervisor S/w will be used to achieve this
- We need to install all the required software in Guest Operating Systems to run our application
- It is an old technique to run the applications
- System performance will become slow in this process
- To overcome the problems of Virtualization, we are going for Containerization concept

---

## Containerization

- It is used to package all the software and application code in one container for execution
- Container will take care of everything which is required to run our application
- We can run the containers in Multiple Machines easily
- Docker is a containerization software
- Using Docker we will create container for our application
- Using Docker we will create image for our application
- Docker images we can share easily to multiple machines
- Using Docker image we can create docker container and we can execute it

### Conclusion

- Docker is a containerization software
- Docker will take care of application and application dependencies for execution
- Application Deployments into multiple environments will become easy if we use Docker containers concept

---

## Environment Setup

### Prerequisites

1. Create Account in AWS Cloud
2. Create Linux Machine using AWS EC2 service (Image: Amazon Linux)
3. Connect to Linux Machine using MobaXterm / Putty
4. Install Docker software in Linux VM using below commands

### Install Docker in Amazon Linux

```bash
sudo yum update -y
sudo yum install docker -y
sudo service docker start

# Add ec2-user to docker group
sudo usermod -aG docker ec2-user

# Restart the session
exit
```

Then press 'R' to restart the session (This is in MobaXterm)

```bash
docker info
```

---

## Docker Commands

### Basic Commands

```bash
# See docker info
docker info

# To see docker images
docker images

# Pulling hello-world docker image
docker pull hello-world

# Running hello-world docker image
docker run hello-world

# Display Running Docker containers
docker ps

# Displaying Running + stopped containers
docker ps -a

# Inspect docker image
docker inspect <image-id>

# Remove Docker image
docker rmi <image-name / image-id>

# Remove docker image forcefully
docker rmi -f <image-name / image-id>

# Stop the container
docker stop <container-id>

# Remove docker container
docker rm <container-id>

# Remove all stopped containers + un-used images + un-used networks
docker system prune -a
```

**Note:** Create account in Docker Hub (https://hub.docker.com/)

---

## Dockerfile

Dockerfile contains set of instructions to build docker image.

- In Dockerfile we will use DSL (Domain Specific Language)
- Docker Engine will read Dockerfile instructions from top to bottom to process

### Dockerfile Keywords

1. FROM
2. MAINTAINER
3. COPY
4. ADD
5. RUN
6. CMD
7. ENTRYPOINT
8. ENV
9. ARG
10. WORKDIR
11. EXPOSE
12. VOLUME
13. USER
14. LABEL

---

### FROM

It represents base image to create our docker image.

**Syntax:**
```dockerfile
FROM java:1.8
FROM python:1.2
FROM mysql:8.5
FROM tomcat:9.5
```

---

### MAINTAINER

It is used to specify docker file author information.

**Syntax:**
```dockerfile
MAINTAINER Ashok <ashok.b@oracle.com>
```

---

### COPY

It is used to copy the files from source to destination while creating docker image.

**Syntax:**
```dockerfile
COPY <SRC> <DESTINATION>
```

**Example:**
```dockerfile
COPY target/sb-api.war /app/tomcat/webapps/sb-api.war
```

---

### ADD

It is used to copy the files from source to destination while creating docker image.

**Syntax:**
```dockerfile
ADD <SRC> <DESTINATION>
ADD <HTTP-URL> <DESTINATION>
```

**Example:**
```dockerfile
ADD <url> /app/tomcat/webapps/sb-api.war
```

#### Difference between COPY and ADD

- **COPY:** It can copy from one path to another path within the same machine
- **ADD:** It can copy from one path to another path & it supports URL also as source

---

### RUN

RUN instructions will execute while creating docker image.

**Syntax:**
```dockerfile
RUN yum install git
RUN yum install maven
RUN git clone <repo-url>
RUN cd <repo-name>
RUN mvn clean package
```

**Note:** We can write multiple RUN instructions, docker engine will process from top to bottom.

---

### CMD

CMD instructions will execute while creating docker container.

Using CMD command we can run our application in container.

**Syntax:**
```dockerfile
CMD sudo start tomcat
CMD java -jar <jar-file>
```

**Note:** We can write multiple CMD instructions but docker engine will process only last CMD instruction.

#### Difference between RUN and CMD

- **RUN** instructions will execute while creating docker image
- **CMD** instructions will execute while creating docker container
- We can write multiple RUN instructions, docker engine will process from top to bottom
- We can write multiple CMD instructions but docker engine will process only last CMD instruction

**Note:** There is no use of writing multiple CMD instructions.

---

### Sample Dockerfile

```dockerfile
FROM ubuntu

MAINTAINER Ashok<ashok.b@oracle.com>

RUN echo "Hi, i am run - 1"
RUN echo "Hi, i am run - 2"
RUN echo "Hi, i am run - 3"

CMD echo "Hi, i am CMD-1"
CMD echo "Hi, i am CMD-2"
CMD echo "Hi, i am CMD-3"
```

### Building and Pushing Docker Image

```bash
# Create a file (filename: Dockerfile)
vi Dockerfile

# Create docker image using Dockerfile
docker build -t <image-name> .

# Login into docker hub account from docker machine
docker login

# Tag docker image
docker tag <image-name> <tagname>
# Example:
docker tag myimg1 ashokit/myimg1

# Push Docker image
docker push <Tag-name>
```

#### Can we use user defined name for Dockerfile?

**Answer:** Yes, we can do it.

```bash
docker build -f <filename> -t <imagename> .
```

---

### ENTRYPOINT

It is used to execute instructions while creating container.

**Syntax:**
```dockerfile
ENTRYPOINT ["echo", "Container Created Successfully"]
ENTRYPOINT ["java", "-jar", "target/springboot.jar"]
```

#### Difference between CMD and ENTRYPOINT

- CMD instructions we can override while creating container
- ENTRYPOINT instructions we can't override

---

### WORKDIR

It is used to specify working directory for image and container.

**Syntax:**
```dockerfile
WORKDIR /app/usr/
```

---

### ENV

ENV is used to set Environment Variables.

**Example:**
```dockerfile
ENV java /etc/softwares/jdk
```

---

### EXPOSE

It is used to specify on which port number our docker container will run.

**Example:**
```dockerfile
EXPOSE 8080
```

---

### ARG

By using ARG we can take dynamic values from CLI.
It is used to remove hard coded values in Dockerfile.

**Example:**
```dockerfile
ARG branch
RUN git clone -b $branch <repo-url>
```

```bash
docker build -t <imagename> --build-arg branch=master .
```

---

### USER

It is used to specify username for creating image/container.

**Example:**
```dockerfile
USER dockeruser
```

---

### VOLUME

It is used to specify docker volume storage location.

Volumes are used for storage purpose.

**Example:**
```dockerfile
VOLUME /data
```

---

### LABEL

It is used to add METADATA to docker objects in key-value format.

**Example:**
```dockerfile
LABEL name="sbi_image"
```

---

## Dockerizing Applications

### Java Applications Dockerization

We can see two types of java applications in the companies:

1. Normal Java Application without Spring Boot
2. Java Application with Spring Boot

**Note:** Spring Boot is a ready-made framework to make java application development simple.

- Normal Java Web Applications will be packaged as war file and war file will be deployed in webserver (Ex: tomcat)
- Spring Boot applications will be packaged as jar file and we need to run the jar file. It will take care of server internally (Embedded Server)

---

### Dockerfile For Java Web Application

**Git Repo:** https://github.com/ashokitschool/maven-web-app.git

```dockerfile
FROM tomcat:8.0.20-jre8

COPY target/app.war /usr/local/tomcat/webapps/app.war

EXPOSE 8080
```

### Working Procedure

```bash
# Connect to Docker Machine
sudo service docker start
sudo yum install git
sudo yum install maven
git clone https://github.com/ashokitschool/maven-web-app.git
cd maven-web-app
mvn clean package
ls -l target
docker build -t maven-web-app .
docker images
docker run -d -p 8080:8080 maven-web-app
docker ps
```

**Note:** Enable 8080 Port in Security Group Inbound Rules (Custom TCP - 8080) which is attached to docker machine.

- Type: Custom TCP
- Port Range: 8080
- Source: Anywhere IPv4

**Access URL:** `http://ec2-vm-publicip:8080/maven-web-app/`

---

### Dockerizing Spring Boot Application

**Git Repo:** https://github.com/ashokitschool/spring-boot-docker-app.git

- Spring Boot app will be packaged as jar file (even if it is web app)
- Spring Boot will have embedded tomcat server to run
- We no need to deploy Spring Boot app in server manually
- We just need to run spring boot app jar file, it will take care of server and deployment

### Spring Boot Application Dockerfile

```dockerfile
FROM openjdk:11

COPY target/app.jar /usr/app/app.jar

WORKDIR /usr/app/

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Working Procedure

```bash
git clone https://github.com/ashokitschool/spring-boot-docker-app.git
cd spring-boot-docker-app
ls -l
mvn clean package
ls -l target
docker images
docker build -t sbapp .
docker images
docker run -d -p 9090:8080 sbapp
docker ps
```

**Note:** Enable 9090 port in security group of docker machine

**Access URL:** `http://ec2-public-ip:9090/`

---

### Python App with Docker

- Python is a general purpose scripting language
- Python programs will have .py extension
- Compilation is not required for Python programs

**Git Repo:** https://github.com/ashokitschool/python-flask-docker-app.git

### Python App Dockerfile

```dockerfile
FROM python:3.6

MAINTAINER Ashok <ashok.b@oracle.com>

COPY . /app/

WORKDIR /app/

EXPOSE 5000

RUN pip install -r requirements.txt

ENTRYPOINT ["python", "app.py"]
```

### Working Procedure

```bash
docker system prune -a
git clone https://github.com/ashokitschool/python-flask-docker-app.git
cd python-flask-docker-app
ls -l
cat Dockerfile
docker build -t python-flask-app .
docker images
docker run -d -p 5000:5000 python-flask-app
docker ps
```

**Note:** As we have mapped container port 5000 to host port 5000, we need to enable 5000 port in security group.

**Access URL:** `http://ec2-public-ip:5000/`

---

### Troubleshooting

```bash
# Print container logs
docker logs <container-id>

# Get into docker container
docker exec -it <container-id> /bin/bash

# To comeout from container use 'exit' command
exit
```

---

### React JS with Docker

- React JS is a JavaScript library
- React JS is used to develop Front end of the application (user interface)
- React JS will use Node Package Manager to install required software

**Git Repo:** https://github.com/ashokitschool/React_App.git

### React JS App Dockerfile

```dockerfile
FROM node:latest
WORKDIR /app
COPY package.json ./
RUN npm install
COPY . .
CMD ["npm", "start"]
```

---

## Docker Network

Network is all about communication.

Docker Network is used to provide isolated network for Docker Container.

### In Docker we have below 3 default networks:

1. bridge
2. host
3. none

### Docker Network Drivers:

1. **Bridge** - This is default network driver
2. **Host**
3. **None**
4. **Overlay** - Docker Swarm
5. **Macvlan**

- **Bridge driver** is recommended driver when we are using standalone container. It will assign one IP for our docker container
- **Host driver** is also used to run standalone container but it will not assign any IP for container
- **None** means no network will be available for docker container
- **Overlay network driver** is used for Orchestration. Docker swarm will use this overlay driver
- **Macvlan network driver** provides physical IP for container

### Docker Network Commands

```bash
# Display Docker Networks
docker network ls

# Create docker network
docker network create ashokit-nw

# Inspect Docker Network
docker network inspect ashokit-nw

# Run Docker container with our network
docker run -d -p 9090:9090 --network ashokit-nw ashokit/spring-boot-rest-api

# Delete Docker network
docker network rm ashokit-nw
```

---

## Docker Compose

Nowadays projects are developing based on Microservices Architecture.

Our application requires multiple containers for execution:
- Frontend app container
- Backend APIs containers (microservices)
- DB containers

Creating multiple containers manually is very difficult and time-taking process.

**Note:** Managing "Multi-Container" based applications is a difficult task.

### Docker Compose is used for Managing Multiple-Containers

- Docker compose is a tool which is used to manage multi-container based applications
- Using Docker compose we can easily setup & deploy multiple containers
- We will use `docker-compose.yml` file to provide containers information to Docker Compose tool
- Docker Compose YML should contain all the information related to containers creation

---

### Docker Compose YML File Structure

```yaml
version:

services:

network:

volumes:
```

**Note:** Docker Compose Default file name is `docker-compose.yml` (we can change it also)

Docker Compose file we will keep in source code repository.

---

### Installing Docker Compose

```bash
# Download docker compose
sudo curl -L "https://github.com/docker/compose/releases/download/1.24.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# Give permission
sudo chmod +x /usr/local/bin/docker-compose

# How to check docker compose is installed or not
docker-compose --version
```

---

### Deploy Spring Boot + MySQL with Docker Compose

```bash
git clone https://github.com/ashokitschool/spring-boot-mysql-docker-compose.git
cd spring-boot-mysql-docker-compose
mvn clean package
docker build -t spring-boot-mysql-app .
docker images
docker-compose up -d
docker-compose ps
```

**Access URL:** `http://ec2-public-ip:8080/`

### Database Operations

```bash
# Check app container logs
docker logs <app-container-name>

# Connect to DB Container
docker exec -it <db-container-name> /bin/bash

# Connect with mysql db using mysql client
mysql -u root -p

# Display databases available in mysql
show databases

# Select db name (sbms is our db name)
use sbms

# Display tables created in database
show tables

# Display table data (book is our tablename)
select * from book;

# Exit from database
exit

# Exit from container
exit
```

---

### Docker Compose Commands

```bash
# Create Containers using Docker Compose
docker-compose up

# Create Containers using different file name
docker-compose -f <filename> up

# Run docker containers in detached mode
docker-compose up -d

# Display containers created by docker compose
docker-compose ps

# Display docker images
docker-compose images

# Check container logs
docker logs -f <container-name>

# Stop & remove docker containers
docker-compose down
```

---

### Sample Docker Compose File

```yaml
version: "3"
services:
  application:
    image: springboot-mysql-app
    ports:
      - 8080:8080
    networks:
      - springboot-db-net
    depends_on:
      - mysqldb
    volumes:
      - /data/springboot-app
  mysqldb:
    image: mysql:5.7
    networks:
      - springboot-db-net
    environment:
      - MYSQL_ROOT_PASSWORD=root
      - MYSQL_DATABASE=sbms
    volumes:
      - /data/mysql
networks:
  springboot-db-net:
```

---

## Stateful Vs Stateless Containers

- **Stateless container:** Data will be deleted after container got deleted
- **Stateful Container:** Data will be maintained permanently

**Note:** Docker Containers are stateless containers (by default)

In the above Spring Boot application, we are using MySQL DB to store the data. When we re-create containers, we lose our data (This is not accepted in realtime).

Even if we deploy latest code or if we re-create containers, we should not lose our data.

To maintain data permanently, we need to make our container as Stateful Container.

To make container as stateful, we need to use Docker Volumes concept.

---

## Docker Volumes

- Volumes are used to persist the data which is generated by Docker container
- Volumes are used to avoid data loss
- Using Volumes we can make container as stateful container

### We have 3 types of volumes in Docker:

1. Anonymous Volume (No Name)
2. Named Volume
3. Bind Mounts

### Docker Volume Commands

```bash
# Display docker volumes
docker volume ls

# Create Docker Volume
docker volume create <vol-name>

# Inspect Docker Volume
docker volume inspect <vol-name>

# Remove Docker Volume
docker volume rm <vol-name>

# Remove all volumes
docker system prune --volumes
```

---

### Making Docker Container Stateful using Bind Mount

Create a directory on host machine:

```bash
mkdir app
```

Map 'app' directory to container in `docker-compose.yml` file like below:

```yaml
version: "3"
services:
  application:
    image: spring-boot-mysql-app
    ports:
      - "8080:8080"
    networks:
      - springboot-db-net
    depends_on:
      - mysqldb
    volumes:
      - /data/springboot-app

  mysqldb:
    image: mysql:5.7
    networks:
      - springboot-db-net
    environment:
      - MYSQL_ROOT_PASSWORD=root
      - MYSQL_DATABASE=sbms
    volumes:
      - ./app:/var/lib/mysql
networks:
  springboot-db-net:
```

### Testing Persistence

```bash
# Start Docker Compose Service
docker-compose up -d

# Access the application and insert data

# Delete Docker Compose service
docker-compose down

# Again start Docker Compose service
docker-compose up -d

# Access application and see data (it should be available)
```

---

## Docker Swarm

- It is a container orchestration software
- Orchestration means managing processes
- Docker Swarm is used to setup Docker Cluster
- Cluster means group of servers
- Docker swarm is embedded in Docker engine (No need to install Docker Swarm Separately)
- We will setup Master and Worker nodes using Docker Swarm cluster
- Master Node will schedule the tasks (containers) and manage the nodes and node failures
- Worker nodes will perform the action (containers will run here) based on master node instructions

---

### Swarm Features

1. Cluster Management
2. Decentralized design
3. Declarative service model
4. Scaling
5. Multi Host Network
6. Service Discovery
7. Load Balancing
8. Secure by default
9. Rolling Updates

---

### Docker Swarm Cluster Setup

Create 3 EC2 instances (ubuntu) & install docker in all 3 instances using below 2 commands:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

**Note:** Enable 2377 port in security group for Swarm Cluster Communications

- 1 - Master Node
- 2 - Worker Nodes

### Connect to Master Machine and execute below command:

```bash
# Initialize docker swarm cluster
sudo docker swarm init --advertise-addr <private-ip-of-master-node>

# Example:
sudo docker swarm init --advertise-addr 172.31.37.100

# Get Join token from master (this token is used by workers to join with master)
sudo docker swarm join-token worker
```

**Note:** Copy the token and execute in all worker nodes with sudo permission

**Example:**
```bash
sudo docker swarm join --token SWMTKN-1-4pkn4fiwm09haue0v633s6snitq693p1h7d1774c8y0hfl9yz9-8l7vptikm0x29shtkhn0ki8wz 172.31.37.100:2377
```

---

### What is docker swarm manager quorum?

If we run only 2 masters, then we can't get High Availability.

**Formula:** `(n-1)/2`

If we take 2 servers:
- `2-1/2 => 0.5` (It can't become master)

If we take 3 servers:
- `3-1/2 => 1` (it can be leader when the main leader is down)

**Note:** Always use odd number for Master machines

---

## Docker Swarm Service

In Docker swarm, we need to deploy our application as a service.

Service is a collection of one or more containers of same image.

### There are 2 types of services in docker swarm:

1. Replica (default mode)
2. Global

### Docker Service Commands

```bash
# Create service
sudo docker service create --name <serviceName> -p <hostPort>:<containerPort> <imageName>

# Example:
sudo docker service create --name java-web-app -p 8080:8080 ashokit/javawebapp
```

**Note:** By default 1 replica will be created

**Access URL:** `http://master-node-public-ip:8080/java-web-app/`

```bash
# Check the services created
sudo docker service ls

# Scale docker service
docker service scale <serviceName>=<no.of.replicas>

# Inspect docker service
sudo docker service inspect --pretty <service-name>

# See service details
sudo docker service ps <service-name>

# Remove one node from swarm cluster
sudo docker swarm leave

# Remove docker service
sudo docker service rm <service-name>
```

---

## Summary

### Topics Covered:

1. What is Application Stack
2. Life without Docker
3. Life with Docker
4. Docker introduction
5. Virtualization vs Containerization
6. Docker Installation in Linux
7. Docker Architecture
8. Docker Terminology
9. Dockerfile & Dockerfile Keywords
10. Writing Dockerfiles
11. Docker image commands
12. Docker container commands
13. Dockerizing Java Spring Boot Application
14. Dockerizing Java Web Application with External Tomcat
15. Dockerizing Python Flask Application
16. Docker Network
17. Monolith Vs Microservices
18. Docker Compose
19. Docker Compose File Creation
20. Docker Volumes
21. Spring Boot with MySQL DB Dockerization using Docker Compose
22. What is Orchestration?
23. Docker Swarm
24. Docker Swarm Cluster Setup
25. Deployed java web app as docker container using swarm cluster

---

## Quick Reference Commands

### Docker Image Commands

```bash
docker info
docker images
docker rmi <imagename>
docker pull <imagename>
docker run <imagename>
docker run -d -p host-port:container-port <image-name>
docker tag <image-name> <image-tag-name>
docker login
docker push <image-tag-name>
```

### Docker Container Commands

```bash
docker ps
docker ps -a
docker stop <container-id>
docker rm <container-id>
docker rm -f <container-id>
docker system prune -a
docker logs <container-id>
docker exec -it <container-id> /bin/bash
```

### Docker Network Commands

```bash
docker network ls
docker network create <network-name>
docker network rm <network-name>
docker network inspect <network-name>
```

### Docker Compose Commands

```bash
docker-compose up -d
docker-compose down
docker-compose ps
docker-compose images
docker-compose stop
docker-compose start
```

### Docker Volume Commands

```bash
docker volume ls
docker volume create <vol-name>
docker volume inspect <vol-name>
docker volume rm <vol-name>
docker system prune --volumes
```

### Docker Swarm Commands

```bash
sudo docker service create --name <service-name> -p 8080:8080 <img-name>
sudo docker service scale <service-name>=<replicas>
sudo docker service ls
sudo docker service rm <service-name>
```

---

## Additional Resources

- **Docker Hub:** https://hub.docker.com/
- **GitHub Repositories:**
  - Java Maven Web App: https://github.com/ashokitschool/maven-web-app.git
  - Spring Boot Docker App: https://github.com/ashokitschool/spring-boot-docker-app.git
  - Python Flask Docker App: https://github.com/ashokitschool/python-flask-docker-app.git
  - React App: https://github.com/ashokitschool/React_App.git
  - Spring Boot MySQL Compose: https://github.com/ashokitschool/spring-boot-mysql-docker-compose.git

---

**End of Document**