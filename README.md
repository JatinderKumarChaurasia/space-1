# space-1

1: Workshop Overview
During this workshop you will learn the finer details of how to create, build, run, and debug a basic Spring Boot app on Kubernetes by doing the following:

Create a basic Spring Boot app
Build a Docker image for the app
Push the image to a Docker registry
Deploy and run the app on Kubernetes
Test the app using port-forwarding and ingress
Use skaffold to iterate easily as you work on your app
Use kustomize to manage configurations across environments
Externalize application configuration using ConfigMaps
Use service discovery for app-to-app communication
Deploy the Spring PetClinic App with MySQL

2: Setup Environment
Workshop Command Execution
This workshop uses action blocks for various purposes, anytime you see such a block with an icon on the right hand side, you can click on it and it will perform the listed action for you.

This is a real environment where action blocks can create real apps and Kubernetes clusters. Please, wait for an action to fully execute before proceeding to the next so the workshop behaves as expected.

Workshop Terminals
Two terminals are included in this workshop, you will mainly use terminal 1, but if it's busy you can use terminal 2 .

Try the action block's bellow

echo "Hi I'm terminal 1"
echo "Hi I'm terminal 2"
Workshop Code Editor
The workshop features a built in code editor you can use by pressing the Editor tab button. Pressing the refresh button in the workshop's UI can help the editor load when switching tabs. The files in this editor automatically save.

The editor takes a few moments to load, please select the Editor tab now to display it, or click on the action block below.

Workshop Console (Ocant)
You will have the ability to inspect your Kubernetes cluster with Octant, an Open Source developer-centric web interface for Kubernetes that lets you inspect a Kubernetes cluster and its applications.

You haven't deployed anything to Kubernetes so there isn't much to display at the moment. When you get to the section 7: Deploying to Kubernetes, you will have a Kubernetes cluster and a Spring Boot app to inspect with Octant.


3: Create a Spring Boot app
Getting Started

First we will create a directory for our app and use start.spring.io to create a basic Spring Boot application.

mkdir demo && cd demo
Download and extract the project from the Spring Initializr

curl https://start.spring.io/starter.tgz -d artifactId=k8s-demo-app -d name=k8s-demo-app -d packageName=com.example.demo -d dependencies=web,actuator -d javaVersion=11 | tar -xzf -
Modify K8sDemoApplication.java and create your Rest controller

First, add the annotations and @RestController

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
Now, add your 'Hello World' rest controller

@GetMapping("/")
public String hello() {
    return "Hello World\n";
}
Your file should look like the following:

package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@SpringBootApplication
public class K8sDemoAppApplication {

    public static void main(String[] args) {
        SpringApplication.run(K8sDemoAppApplication.class, args);
    }

    @GetMapping("/")
    public String hello() {
        return "Hello World\n";
    }
}

4: Run the App
In a terminal window run the following command. The application will start on port 8080.

./mvnw spring-boot:run
Test The App

Make an HTTP request to http://localhost:8080 in another terminal

curl http://localhost:8080
Test Spring Boot Actuator

Spring Boot Actuator adds several other endpoints to our app. By default Spring Boot will expose a health and info endpoint.

curl localhost:8080/actuator | jq .
Your output will be similar to this.

{
  "_links": {
    "self": {
      "href": "http://localhost:8080/actuator",
      "templated": false
    },
    "health": {
      "href": "http://localhost:8080/actuator/health",
      "templated": false
    },
    "info": {
      "href": "http://localhost:8080/actuator/info",
      "templated": false
    }
}
Be sure to stop the Java process before continuing on or else you might get port binding issues since Java is using port 8080

<ctrl+c>
