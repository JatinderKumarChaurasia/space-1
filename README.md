# space-1

23: Kustomize
So far, we have deployed our application to a single Kubernetes environment. Most likely, we would want to deploy it to several environments (e.g. dev, test, prod). We could create separate Kubernetes deployment manifests for each environment, or we could look to tools to help de-duplicate and manage environment-specific differences in our configuration.

In this section we will look at one such tool: Kustomize allows us to customize deployments to different environments.

We can start with a base set of resources and then apply customizations on top of those.

Features

Allows easier deployments to different environments/providers
Allows you to keep all the common properties in one place
Generate configuration for specific environments
No templates, no placeholder spaghetti, no environment variable overload


24: Getting Started with Kustomize
Create a new directory in the root of our project called kustomize/base

mkdir -p kustomize/base
Move the deployment.yaml and service.yaml from the k8s directory into kustomize/base

mv k8s/* kustomize/base
Delete the k8s directory since we no longer need it

rm -Rf k8s

25: kustomization.yaml
kustomization.yaml

In kustomize/base create a new file called kustomization.yaml and add the following to it.
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:    
- service.yaml
- deployment.yaml
- ingress.yaml
NOTE: Optionally, you can now remove all the labels and annotations in the metadata of objects and specs inside objects. Kustomize adds default values that link objects to each other (e.g. to link a service to a deployment). If there is only one of each in your manifest then it will pick something sensible.


26: Customizing Our Deployment
Lets imagine we want to deploy our app to a QA environment, but in this environment we want to have two instances of our app running. To do this we can create a new Kustomization for this environment which specifies the number of instances we want deployed.

Create a new directory called qa under the kustomize directory

mkdir -p kustomize/qa
Create a new file in kustomize/qa called update-replicas.yaml, this is where we will specify that we want 2 replicas running

apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-demo-app
spec:
  replicas: 2
Create a new file called kustomization.yaml in kustomize/qa and add the following to it

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../base

patchesStrategicMerge:
- update-replicas.yaml
Here we tell Kustomize that we want to patch the resources from the base directory with the update-replicas.yaml file. Notice that in update-replicas.yaml we are just updating the properties we care about, in this case the replicas.


27: Running Kustomize
You can use kustomize build to build the customizations and produce all of the YAML to deploy our application. This command will return the generated YAML as output in the terminal.

To build the base profile for our application run the following command.

kustomize build ./kustomize/base
When we build the QA customization the replicas property is updated to 2.

kustomize build ./kustomize/qa
...
spec:
  replicas: 2
...

28: Piping to Kustomize
We can pipe the output from kustomize into kubectl in order to use the generated YAML to deploy our app to Kubernetes

kustomize build kustomize/qa | kubectl apply -f -
If you are watching the pods in your Kubernetes namespace you will see two pods created instead of one because we generated the YAML using the QA customization.

watch -n 1 kubectl get all
Every 1.0s: kubectl get all                                 eduk8s-labs-w01-s010-669db9789f-dkdd5: Mon Feb  3 12:00:04 2020

NAME                                READY   STATUS    RESTARTS   AGE
pod/k8s-demo-app-647b8d5b7b-r2999   1/1     Running   0          83s
pod/k8s-demo-app-647b8d5b7b-x4t54   1/1     Running   0          83s

NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/k8s-demo-app   ClusterIP   10.100.200.200   <none>        80/TCP    84s

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/k8s-demo-app   2/2     2            2           84s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/k8s-demo-app-647b8d5b7b   2         2         2       84s
To exit the watch command

<ctrl+c>

29: Clean Up
Before continuing clean up your Kubernetes environment
kustomize build kustomize/qa | kubectl delete -f -

