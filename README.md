# space-1

32: Externalized Configuration
One of the 12 factors(https://12factor.net/config) for cloud native apps is to externalize configuration
Kubernetes provides support for externalizing configuration via ConfigMaps and Secrets
We can create a ConfigMap or secret easily using kubectl. For example

kubectl create configmap log-level --from-literal=LOGGING_LEVEL_ORG_SPRINGFRAMEWORK=DEBUG
kubectl get configmap log-level -o yaml
apiVersion: v1
data:
  LOGGING_LEVEL_ORG_SPRINGFRAMEWORK: DEBUG
kind: ConfigMap
metadata:
  creationTimestamp: "2020-02-04T15:51:03Z"
  name: log-level
  namespace: eduk8s-labs-w02-s204
  resourceVersion: "2145271"
  selfLink: /api/v1/namespaces/default/configmaps/log-level
  uid: 742f3d2a-ccd6-4de1-b2ba-d1514b223868

33: Using ConfigMaps in Our Apps
There are a number of ways to consume the data from ConfigMaps in our apps

Perhaps the easiest is to use the data as environment variables

To do this we need to change our deployment.yaml in kustomize/base

Add the envFrom properties from the previous module which reference our ConfigMap log-level

envFrom:
  - configMapRef:
      name: log-level
Update the deployment by running skaffold dev (so we can stream the logs)

skaffold dev
If everything worked correctly you should see much more verbose logging in your console

Be sure to kill the skaffold process before continuing

<ctrl+c>


  
