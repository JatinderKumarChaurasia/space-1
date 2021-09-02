# space-1

5: Containerize the App
alt_text

Building A Container

Spring Boot 2.3.x and newer can build a container for you without the need for any additional plugins or files
To do this use the Spring Boot Build plugin goal build-image
./mvnw spring-boot:build-image
Running docker images will allow you to see the built container
docker images
Run The Container

docker run --name k8s-demo-app -p 8080:8080 k8s-demo-app:0.0.1-SNAPSHOT
Test The App Responds

curl http://localhost:8080
Be sure to stop the Docker container before continuing.

docker rm -f k8s-demo-app 

6: Putting The Container In A Registry
Up until this point the container only lives on your machine
It is useful to instead place the container in a registry
In a real life scenario this allows others to use the container
Docker Hub is a popular public registry
Private registries exist as well. In this lab you will be using a private registry on localhost.
Run The Build And Deploy The Container

For convenience, the address of the local private docker registry for this lab is saved in an environment variable. You can see it by running the following command.
echo $REGISTRY_HOST
Run the following Maven command to re-build the image, this time including the registry address in the image name by setting the property spring-boot.build-image.imageName.
./mvnw spring-boot:build-image -Dspring-boot.build-image.imageName=$REGISTRY_HOST/apps/demo
Now we can push the container to the local container registry

docker push $REGISTRY_HOST/apps/demo
If you query the registry you should now see the image

skopeo list-tags docker://$REGISTRY_HOST/apps/demo
You should see a print out like the this.

{
    "Repository": "lab-workshop-1-w01-s001-registry.192.168.64.2.nip.io/apps/demo",
    "Tags": [
        "latest"
    ]
}

7: Deploying to Kubernetes
With our container built and deployed to a registry you can now run this container on Kubernetes
Deployment Descriptor

Kubernetes uses YAML files to provide a way of describing how the app will be deployed to the platform
You can write these by hand using the Kubernetes documentation as a reference
Or you can have Kubernetes generate them for you using kubectl
The --dry-run flag allows us to generate the YAML without actually deploying anything to Kubernetes
Lets create a directory to put the files we will need to deploy our application to Kubernetes

mkdir k8s
The first file we need is a deployment descriptor. Execute the following command to create the file

kubectl create deployment k8s-demo-app --image eduk8s-labs-w02-s204-registry.springone2021-49e5522.tanzu-labs.esp.vmware.com/apps/demo -o yaml --dry-run=client > ~/demo/k8s/deployment.yaml
The resulting k8s/deployment.yaml should look like this. You can switch to the Editor tab to compare with the file you just created.
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: k8s-demo-app
  name: k8s-demo-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: k8s-demo-app
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: k8s-demo-app
    spec:
      containers:
      - image: eduk8s-labs-w02-s204-registry.springone2021-49e5522.tanzu-labs.esp.vmware.com/apps/demo
        name: k8s-demo-app
        resources: {}
status: {}
Service Descriptor

A service acts as a load balancer for the pod created by the deployment descriptor
If we want to be able to scale pods than we want to create a service for those pods
kubectl create service clusterip k8s-demo-app --tcp 80:8080 -o yaml --dry-run=client > ~/demo/k8s/service.yaml
The resulting service.yaml should look similar to this. Again, you can switch to the Editor tab to compare with the file you just created.
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: k8s-demo-app
  name: k8s-demo-app
spec:
  ports:
  - name: 80-8080
    port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: k8s-demo-app
  type: ClusterIP
status:
  loadBalancer: {}
Apply The Deployment and Service YAML

The deployment and service descriptors have been created in the /k8s directory
Since we used the --dry-run flag to generate them, they have not yet been applied to Kubernetes
Apply these now to get everything running
You can watch as the pods and services get created
To apply your manifest files and get the app running, run the following command

kubectl apply -f ~/demo/k8s 
Execute the following watch command to see the deployment progress in Kubernetes

watch -n 1 kubectl get all
You should see something like the following:

NAME                               READY   STATUS    RESTARTS   AGE
pod/k8s-demo-app-d6dd4c4d4-7t8q5   1/1     Running   0          68m

NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/k8s-demo-app   ClusterIP   10.100.200.243   <none>        80/TCP    68m

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/k8s-demo-app   1/1     1            1           68m

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/k8s-demo-app-d6dd4c4d4   1         1         1       68m
watch is a useful command line tool that you can install on Linux and OSX. All it does is continuously execute the command you pass it. You can just run the kubectl command specified after the watch command but the output will be static as opposed to updating constantly.

Terminate the watch process before to continuing.

<ctrl+c>

8: Testing the App
The service is assigned a cluster IP, which is only accessible from inside the cluster.
To see this run the next command.

kubectl get service/k8s-demo-app
You should have gotten output like the following.

NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/k8s-demo-app   ClusterIP   10.100.200.243   <none>        80/TCP    68m
To access the app we can use kubectl port-forward to forward requests on port 8080 locally to port 80 on the service running in Kubernetes
kubectl port-forward service/k8s-demo-app 8080:80
Now we can curl localhost:8080 and it will be forwarded to the service in the cluster
curl http://localhost:8080
Congrats you have deployed your first app to Kubernetes 🎉

Be sure to stop the kubectl port-forward process before moving on

<ctrl+c>

9: Exposing the Service
If we want to expose the service outside of the cluster we can use an Ingress.
An Ingress is an API object that defines rules which allow external access to services in a cluster. An Ingress controller fulfills the rules set in the Ingress.

Next, you will go create ingress.yaml so you can access your app outside of the cluster.

Click below to create your ingress.yaml.

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: k8s-demo-app
  labels:
    app: k8s-demo-app
spec:
  rules:
  - host: k8s-demo-app-eduk8s-labs-w02-s204.springone2021-49e5522.tanzu-labs.esp.vmware.com
    http:
      paths:
      - path: "/"
        pathType: Prefix
        backend:
          service:
            name: k8s-demo-app
            port: 
              number: 80
Now, apply the ingress.yaml.

kubectl apply -f ~/demo/k8s

10: Testing the Public Ingress
Use the following command to view the host and IP address.

The -w option of kubectl lets you watch a single Kubernetes resource.

kubectl get ingress k8s-demo-app -w
You should see output like the following. Depending on the ingress being used in the Kubernetes cluster you might need to wait for an IP address to be assigned to the ingress. If you are running this workshop in a Cloud environment (Google, Amazon, Azure etc.), or locally in a cluster like Minikube, you may need to wait for an IP address to be assigned. In the case of a cloud environment, Kubernetes will assign the service an external IP. In this hosted lab environment, only the host name will be provided, so you will not see an IP address assigned.

NAME           CLASS    HOSTS                                                                          ADDRESS                                                                   PORTS   AGE
k8s-demo-app   <none>   k8s-demo-app-eduk8s-labs-w01-s010.s1t-prod-31a4032.tanzu-labs.esp.vmware.com   a42e0185b95f34449943ea4c560be1d3-1763013971.us-east-2.elb.amazonaws.com   80      90s
Exit from the watch command

<ctrl+c>
Test the ingress configuration execute the following command.

curl k8s-demo-app-eduk8s-labs-w02-s204.springone2021-49e5522.tanzu-labs.esp.vmware.com

11: Best Practices
Kubernetes uses two probes to determine if the app is ready to accept traffic and whether the app is alive
If the liveness probe does not return a 200 Kubernetes will restart the Pod
If the readiness probe does not return a 200 no traffic will be routed to it
Spring Boot has a built in set of endpoints from the Actuator module that fit nicely into these use cases
The /health/readiness endpoint can be used for the readiness probe
The /health/liveness endpoint can be used for the liveness probe
The /health/readiness and /health/liveness endpoints are only available in Spring Boot 2.3.x. The/health and /info endpoints are reasonable starting points in earlier versions.

12: Add Readiness and Liveness Probes
Add The Readiness Probe

readinessProbe:
  httpGet:
    port: 8080
    path: /actuator/health/readiness
Add The Liveness Probe

livenessProbe:
  httpGet:
    port: 8080
    path: /actuator/health/liveness
    
 
 13: Graceful Shutdown
Due to the asynchronous way Kubnernetes shuts down applications there is a period of time when requests can be sent to the application while an application is being terminated.
To deal with this we can configure a pre-stop sleep to allow enough time for requests to stop being routed to the application before it is terminated.
Add a preStop command to the podspec of your deployment.yaml
lifecycle:
  preStop:
    exec:
      command:
        - sh
        - '-c'
        - sleep 10

14: Handling In Flight Requests
Our application could also be handling requests when it receives the notification that it needs to shut down.
In order for us to finish processing those requests before the application shuts down we can configure a “grace period” in our Spring Boot applicaiton.
Open application.properties in /src/main/resources and add
server.shutdown=graceful
There is also a spring.lifecycle.timeout-per-shutdown-phase (default 30s).

server.shutdown is only available begining in Spring Boot 2.3.x

15: Update The Container & Apply The Updated Deployment YAML
Let’s update the pom.xml to configure the image name explicitly:

<spring-boot.build-image.imageName>eduk8s-labs-w02-s204-registry.springone2021-49e5522.tanzu-labs.esp.vmware.com/apps/demo</spring-boot.build-image.imageName>
Then we can build and push a new image to our repository:

./mvnw clean spring-boot:build-image
docker push eduk8s-labs-w02-s204-registry.springone2021-49e5522.tanzu-labs.esp.vmware.com/apps/demo
When we apply our resources again the old container will be terminated and a new one will be deployed with the new changes.

kubectl apply -f ~/demo/k8s
You can run the following command to watch Kubernetes terminate the old container and redeploy a new one in real time. Notice the status of the older pod changing from Running to Terminating before the old pod disappears.

watch -n 1 kubectl get all
You can still test with curl.

curl k8s-demo-app-eduk8s-labs-w02-s204.springone2021-49e5522.tanzu-labs.esp.vmware.com
To exit running process in terminal one.

<ctrl+c>

16: Cleaning-up
Before we move on to the next section lets clean up everything we deployed
kubectl delete -f ~/demo/k8s

