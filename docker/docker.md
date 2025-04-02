# Docker Guide

- [Docker Guide](#docker-guide)
  - [What is Docker?](#what-is-docker)
    - [What is a Docker Image?](#what-is-a-docker-image)
    - [What is a Docker Container?](#what-is-a-docker-container)
    - [Why Use Docker?](#why-use-docker)
    - [Setting Up Docker](#setting-up-docker)
  - [Creating a Dockerfile](#creating-a-dockerfile)
    - [Example Dockerfile](#example-dockerfile)
    - [Building Docker Images](#building-docker-images)
    - [Running Docker Containers](#running-docker-containers)
    - [Managing Docker Containers](#managing-docker-containers)
  - [What is Docker Compose?](#what-is-docker-compose)
    - [Example Docker Compose File](#example-docker-compose-file)
  - [Docker Swarm](#docker-swarm)

<br>

## What is Docker?

Docker is a platform that allows developers to automate the deployment of applications inside lightweight, portable containers. These containers can run on any machine that has Docker installed, making it easy to develop, ship, and run applications consistently across different environments.

<br>

### What is a Docker Image?

A Docker image is a lightweight, standalone, and executable package that includes everything needed to run a piece of software, including the code, runtime, libraries, environment variables, and configuration files. Images are read-only and can be used to create containers.

<br>

### What is a Docker Container?

A Docker container is a running instance of a Docker image. 

<br>

### Why Use Docker?

- **Isolation**: Each container runs in its own environment, ensuring that applications do not interfere with each other.
- **Portability**: Containers can run on any system that supports Docker, regardless of the underlying OS or hardware.
- **Scalability**: Docker makes it easy to scale applications up or down by adding or removing containers as needed.
- **Efficiency**: Containers share the host OS kernel, making them lightweight and fast to start compared to traditional virtual machines.
- **Version Control**: Docker images can be versioned, allowing you to roll back to previous versions of your application easily.
- **Ecosystem**: Docker has a rich ecosystem of tools and services, including Docker Hub for sharing images, Docker Compose for multi-container applications, and Docker Swarm/Kubernetes for orchestration.

<br>

### Setting Up Docker

1. **Install Docker**: Follow the official [Docker installation guide](https://docs.docker.com/get-docker/) for your operating system.
2. **Verify Installation**: Run `docker --version` to check if Docker is installed correctly.
3. **Run a Test Container**: Use the command `docker run hello-world` to verify that Docker is working properly. This command downloads a test image and runs it in a container, displaying a message if successful.

<br>

## Creating a Dockerfile

A Dockerfile is a text file that contains a set of instructions for building a Docker image. It specifies the base image, application code, dependencies, and any other configurations needed to create the image. Dockerfiles are used to automate the process of creating Docker images, ensuring consistency and reproducibility.

[Dockerfile](/docker/Dockerfile)

<br>

### Example Dockerfile

```dockerfile
# Use the official Node.js image as the base image
FROM node:22 AS builder

# Set the working directory in the container
WORKDIR /app

# Copy package.json and package-lock.json to the working directory
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy the rest of the application code to the working directory
COPY . .

# Build the application
RUN npm run build

# Use a lightweight image for the final build
FROM node:22-alpine

# Set the working directory in the container
WORKDIR /app

# Copy the built assets from the builder stage
COPY --from=builder /app/build ./build
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package*.json ./
COPY --from=builder /app/public ./public

# Expose the application port
EXPOSE 3000

# Instructions to run the application when the container starts
ENTRYPOINT ["npm", "start"]
```

<br>

### Building Docker Images

To build a Docker image from a **Dockerfile**, you can use the `docker build` command. The command takes the path to the directory containing the Dockerfile and builds the image based on the instructions in the file.

This builds a Docker image from the Dockerfile in the current directory, tagging it with the name `my-node-app`. The `-t` flag allows you to specify a name and optionally a tag for the image. The `.` at the end specifies the build context, which is the current directory in this case.

```bash
docker build -t my-node-app .
```

<br>

### Running Docker Containers

**To run a Docker container**, you can use the `docker run` command followed by the image name. For example, to run a container from the official Node.js image, you can use:

```bash
docker run -d -p 3000:3000 --name my-node-app node:22
```

<br>

The image name is node:22 which is the official Node.js image from Docker Hub. The `-d` flag runs the container in detached mode, meaning it runs in the background. The `-p` flag maps port 3000 on the host to port 3000 in the container, allowing you to access the application from your host machine on port 3000. The `--name` flag gives the container a name (my-node-app) for easier management.

<br>

---

**To run a container you built**, you need to specify the image name you used when building it. For example, if you built an image locally with the name `my-node-app`, you can run it with:

```bash
docker run -d -p 3000:3000 --name my-node-app my-node-app
```

<br>

---

**Or if you want to run a container from a external image**, you first need to pull the image from a Docker registry (like Docker Hub) using the `docker pull` command followed by the image name. For example, if you have an image named `myname/my-node-app:latest` on Docker Hub, you can pull it with:

```bash
docker pull myname/my-node-app:latest
```

Then, you can run a container from the pulled image:

```bash
docker run -d -p 3000:3000 --name my-node-app myname/my-node-app:latest
```

<br>

---

**To have a container automatically restart** unless it is explicitly stopped, you can use the `--restart unless-stopped` flag with the `docker run` command.

```bash
docker run -d -p 3000:3000 --name my-node-app --restart unless-stopped <image_name>
```

<br>

To add this flag to an existing container, you can use the `docker update` command:

```bash
docker update --restart unless-stopped <container_id>
```

<br>

### Managing Docker Containers

**List Running Containers**: To see a list of all running containers, use the command:

  ```bash
  docker ps
  ```

<br>

**List All Containers**: To see a list of all containers (running and stopped), use the command:

  ```bash
  docker ps -a
  ```

<br>

**Stop a Container**: To stop a running container, use the command:

  ```bash
  docker stop <container_id>
  ```

<br>

**Start a Container**: To start a stopped container, use the command:

  ```bash
  docker start <container_id>
  ```

<br>

**Remove a Container**: To remove a stopped container, use the command:

  ```bash
  docker rm <container_id>
  ```

<br>

**Remove an Image**: To remove a Docker image, use the command:

  ```bash
  docker rmi <image_id>
  ```

<br>

**View Container Logs**: To view the logs of a running container, use the command:

  ```bash
  docker logs <container_id>
  ```

<br>

**Execute a Command in a Running Container**: To execute a command inside a running container, use the command:

  ```bash
  docker exec -it <container_id> <command>
  ```

<br>

## What is Docker Compose?

Docker Compose is a tool for defining and running multi-container Docker applications. It allows you to define a multi-container application in a single YAML file, specifying the services, networks, and volumes needed for the application. With Docker Compose, you can easily manage the lifecycle of your application, including starting, stopping, and scaling services.

<br>

### Example Docker Compose File

```yaml
# Defines the version of the Docker Compose file format
version: '3.8'

# Defines the services that make up the application
services:

  # Service for the application
  # This service builds the application from the Dockerfile
  app:

    # Build the application using the Dockerfile in the current directory
    build:
      context: .
      dockerfile: Dockerfile
    
    # Set the port mapping for the application
    ports:
      - "3000:3000"

    # Set the environment variables for the application
    environment:
      - NODE_ENV=production
  
  # Service for the PostgreSQL database
  # This service uses the official PostgreSQL image
  db:

    # Use the official PostgreSQL image
    image: postgres:latest

    # Set the environment variables for the database
    environment:
      - POSTGRES_USER=myuser
      - POSTGRES_PASSWORD=mypassword
      - POSTGRES_DB=mydb

    # Set volumes for persistent data storage 
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  # Define a named volume for the database
  db_data:
```

<br>

## Docker Swarm

> [!NOTE]
> Not covered in this guide yet.