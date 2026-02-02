# Learn Microservices with Spring Boot - (Spring Cloud and Kafka and Kubernetes)
This repository contains the source code of the practical use case described in the book [Learn Microservices with Spring Boot 3 (3rd Edition)](https://link.springer.com/book/10.1007/978-1-4842-9757-5).
And I made some changes to logback kafka appender to add headers https://github.com/danielwegener/logback-kafka-appender and add micrometer zipkin trace support.

## Features

The figure below shows a high-level overview of the final version of our system.

![Logical View - Chapter 8 (Final)](resources/microservice_patterns-Config-Server-1.png)

The main concepts included in this project are:

* Why do we need Centralized Logs and Distributed tracing?
* Why would I create Docker images for my applications?
* Building a simple logger application with Spring Boot and Kafka using logback-kafka-appender.
* Distributed traces with Micrometer and Zipkin.
* Building Docker images for Spring Boot applications with Cloud Native Buildpacks.
* Container Platforms, Application Platforms, and Cloud Services.

## Running the app

## Playing with Kind kubernetes cluster

  1. create a file kind-config.yaml with content below and create Kind cluster

	kind: Cluster
	apiVersion: kind.x-k8s.io/v1alpha4
	name: kind
	nodes:
	- role: control-plane
	  extraPortMappings:
	  - containerPort: 80
		hostPort: 80
		protocol: TCP
	  - containerPort: 443
		hostPort: 443
		protocol: TCP
	containerdConfigPatches:
	- |-
	  [plugins."io.containerd.grpc.v1.cri".dns]
		servers = ["8.8.8.8", "1.1.1.1"]

  kind create cluster --config kind-config.yaml
  
  2.   install ingress-nginx
  kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.6/deploy/static/provider/kind/deploy.yaml
  

### Building the images yourself

First, build the application images with:

```bash
multiplication$ ./mvnw spring-boot:build-image
gamification$ ./mvnw spring-boot:build-image
gateway$ ./mvnw spring-boot:build-image
logs$ ./mvnw spring-boot:build-image
```

or

```bash
multiplication$ docker build -t  multiplication:0.0.1-SNAPSHOT .
gamification$ docker build -t  gamification:0.0.1-SNAPSHOT .
gateway$ docker build -t  gateway:0.0.1-SNAPSHOT .
logs$ docker build -t  logs:0.0.1-SNAPSHOT .
```

And the UI server (first you have to build it with `npm run build`):

```bash
challenges-frontend$ npm install
challenges-frontend$ npm run build
challenges-frontend$ docker build -t challenges-frontend:1.0-SNAPSHOT .
```

## Load local docker images to Kind

```bash
kind load docker-image "gateway:0.0.1-SNAPSHOT" --name kind
kind load docker-image "gamification:0.0.1-SNAPSHOT" --name kind
kind load docker-image "multiplication:0.0.1-SNAPSHOT" --name kind
kind load docker-image "logs:0.0.1-SNAPSHOT" --name kind
kind load docker-image "challenges-frontend:1.0-SNAPSHOT" --name kind


kind load docker-image bitnami/kafka:latest --name kind     //official images can ignore
kind load docker-image openzipkin/zipkin:latest --name kind //official images can ignore
```

See the figure below for a diagram showing the container view.

![Container View](resources/microservice_patterns-View-Containers.png)

Once the backend and the frontend are started, you can navigate to `http://localhost:80` in your browser and start resolving multiplication challenges.


And you'll get two instances of each of these services with proper Load Balancing and Service Discovery.

