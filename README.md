# space-1

38: Service Discovery
Kubernetes makes it easy to make requests to other services
Each service has a DNS entry in the container of the other services allowing you to make requests to that service using the service name
For example, if there is a service called k8s-workshop-name-service deployed we could make a request from the k8s-demo-app just by making an HTTP request to http://k8s-workshop-name-service


39: Deploying Another App
In order to save time in the following modules we will use an existing app that returns a random name
The image for this service resides in Docker Hub (a public container registry)
To make things easier we placed a Kustomization file in the GitHub repo that we can reference from our own Kustomization file to deploy the app to our cluster

40: Modify `kustomization.yaml`
In your k8s-demo-app’s kustomize/base/kustomization.yaml add a new resource pointing to the new app’s kustomize directory

- https://github.com/ryanjbaxter/k8s-spring-workshop/name-service/kustomize/base

41: Making A Request To The Service
Delete previous implementation of the hello method in K8sDemoAppApplication.java

 sed -i '19d' ~/demo/src/main/java/com/example/demo/K8sDemoAppApplication.java
Modify the hello method of K8sDemoApplication.java to make a request to the new service.

First add the appropriate imports for RestTemplate and RestTemplateBuilder.

import org.springframework.web.client.RestTemplate;
import org.springframework.boot.web.client.RestTemplateBuilder;
Instantiate a new RestTemplate instance.

private RestTemplate rest = new RestTemplateBuilder().build();
Finally use the RestTemplate to make a GET request to the k8s-workshop-name-service.

String name = rest.getForObject("http://k8s-workshop-name-service", String.class);
return "Hola " + name;
Notice the hostname of the request we are making matches the service name in our service.yaml file
To verify your work, this is what your K8sDemoAppApplication.java file should look like.

package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;
import org.springframework.boot.web.client.RestTemplateBuilder;

@RestController
@SpringBootApplication
public class K8sDemoAppApplication {
private RestTemplate rest = new RestTemplateBuilder().build();

    public static void main(String[] args) {
        SpringApplication.run(K8sDemoAppApplication.class, args);
    }

    @GetMapping("/")
    public String hello() {
        String name = rest.getForObject("http://k8s-workshop-name-service", String.class);
        return "Hola " + name;
    }
}


42: Testing the App
Modify the skaffold.yaml file to specify the port to forward to. This is optional, Skaffold will allocate a random port. but for the sake of everyone running through this workshop and having some consistency we specify the port in our Skaffold configuration.

- resourceType: service
  resourceName: k8s-workshop-name-service 
  port: 80
  localPort: 4504
Deploy the apps using Skaffold

skaffold dev --port-forward
This should deploy both the k8s-demo-app and the name-service app
kubectl get all
Your output will be similar to:

NAME                                             READY   STATUS    RESTARTS   AGE
pod/k8s-demo-app-5b957cf66d-w7r9d                1/1     Running   0          172m
pod/k8s-workshop-name-service-79475f456d-4lqgl   1/1     Running   0          173m

NAME                                TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)        AGE
service/k8s-demo-app                LoadBalancer   10.100.200.102   35.238.231.79   80:30068/TCP   173m
service/k8s-workshop-name-service   ClusterIP      10.100.200.16    <none>          80/TCP         173m

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/k8s-demo-app                1/1     1            1           173m
deployment.apps/k8s-workshop-name-service   1/1     1            1           173m

NAME                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/k8s-demo-app-5b957cf66d                1         1         1       172m
replicaset.apps/k8s-demo-app-fd497cdfd                 0         0         0       173m
replicaset.apps/k8s-workshop-name-service-79475f456d   1         1         1       173m
Because we deployed two services and supplied the -port-forward flag, Skaffold will forward two ports

Port forwarding service/k8s-demo-app in namespace lab-workshop-1-w01-s002, remote port 80 -> address 127.0.0.1 port 4503
Port forwarding service/k8s-workshop-name-service in namespace lab-workshop-1-w01-s002, remote port 80 -> address 127.0.0.1 port 4504
Test the name service
curl localhost:4504
Hitting the service multiple times will return a different name

Test the k8s-demo-app which should now make a request to the name-service
curl localhost:4503
Making multiple requests should result in different names coming from the name-service

curl localhost:4503
Stop the Skaffold process to clean everything up before moving to the next step

<ctrl+c>

43: Running The PetClinic App
The PetClinic app is a popular demo app which leverages several Spring technologies
Spring Data (using MySQL)
Spring Caching
Spring WebMVC
alt_text


44: Deploying PetClinic
We have a Kustomization that we can use to easily get it up and running
kustomize build https://github.com/dsyer/docker-services/layers/samples/petclinic?ref=HEAD | kubectl apply -f -
The above kustomize build command may take some time to complete. You can watch the pod status to know once everything is ready.

watch kubectl get all
Once the pod is up and running, you can stop the watch command.

To exit the watch command

<ctrl+c>
Add an ingress rule to expose the service

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: petclinic-app
  labels:
    app: petclinic-app
spec:
  rules:
  - host: petclinic-app-eduk8s-labs-w02-s204.springone2021-49e5522.tanzu-labs.esp.vmware.com
    http:
      paths:
      - path: "/"
        pathType: Prefix
        backend:
          service:
            name: petclinic-app
            port: 
              number: 80
Use the following command to apply the new ingress rule to the cluster

 kubectl apply -f ~/demo/petclinic-ingress.yaml
Depending on the ingress being used in the Kubernetes cluster you might need to wait for an IP address to be assigned to the ingress. For this lab, if you are using the hosted version you will not see an IP address assigned. If you are running the workshop locally on a Kubernetes cluster like Minikube you will need to wait for an IP address to be assigned.

Use the following command to find the hostname for the ingress rule and optionally watch for an IP address to be assigned to the ingress rule.

kubectl get ingress -w
You can open the host name in your browser or click the action below to open the Pet Clinic dashboard in the workshop.

To use the app you can go to “Find Owners”, add yourself, and add your pets
All this data will be stored in the MySQL database
To exit the watch command

<ctrl+c>

45: Dissecting PetClinic
Here’s the kustomization.yaml that you just deployed:
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
- ../../mysql
namePrefix: petclinic-
transformers:
- ../../mysql/transformer
- ../../actuator
- ../../prometheus
images:
  - name: dsyer/template
    newName: dsyer/petclinic
configMapGenerator:
  - name: env-config
    behavior: merge
    literals:
      - SPRING_CONFIG_LOCATION=classpath:/,file:///config/bindings/mysql/meta/
      - MANAGEMENT_ENDPOINTS_WEB_BASEPATH=/actuator
      - DATABASE=mysql
The relative paths ../../* are all relative to this file. Clone the repository to look at those: git clone https://github.com/dsyer/docker-services and look at layers/samples/petclinc/kustomization.yaml.
Clone the repository for the kustomization.yaml.

git clone https://github.com/dsyer/docker-services
Look at ~/docker-services/layers/samples/petclinic/kustomization.yaml.

base
Important features:
base: a generic Deployment and Service with a Pod listening on port 8080
mysql: a local MySQL Deployment and Service. Needs a PersistentVolume so only works on Kubernetes clusters that have a default volume provider
transformers: patches to the basic application deployment. The patches are generic and could be shared by multiple different applications.
env-config: the base layer uses this ConfigMap to expose environment variables for the application container. These entries are used to adapt the PetClinic to the Kubernetes environment.

46: Workshop Summary
During this workshop you learned the finer details of how to create, build, run, and debug a basic Spring Boot app on Kubernetes by doing the following:

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
