# space-1

30: Using Kustomize with Skaffold
Using Kustomize With Skaffold

Currently our Skaffold configuration uses kubectl to deploy our artifacts, but we can change that to use kustomize.

First, delete your previous skaffold.yaml

rm skaffold.yaml
Create a new skaffold.yaml using the following content.

apiVersion: skaffold/v2beta5
kind: Config
metadata:
  name: k-s-demo-app
build:
  artifacts:
  - image: eduk8s-labs-w02-s204-registry.springone2021-49e5522.tanzu-labs.esp.vmware.com/apps/demo
    buildpacks:
      builder: gcr.io/paketo-buildpacks/builder:base-platform-api-0.3
      dependencies:
        paths:
          - src
          - pom.xml
deploy:
  kustomize:
    paths: ["kustomize/base"]
profiles:
  - name: qa
    deploy:
      kustomize:
        paths: ["kustomize/qa"]
portForward:
- resourceType: service
  resourceName: k8s-demo-app 
  port: 80
  localPort: 4503
Notice now the deploy property has been changed from kubectl to now use kustomize
Also notice that we have a new profiles section allowing us to deploy our QA configuration using Skaffold

31: Testing Skaffold + Kustomize
If you run skaffold without specifying a profile parameter Skaffold will use the deployment configuration from kustomize/base.

skaffold dev --port-forward
Terminate the currently running Skaffold process.

<ctrl+c>
Run the following command specifying the qa profile to test out the QA deployment.

skaffold dev -p qa --port-forward
Test your deployment freely. You can validate that two pods are created when the qa profile is specified.

kubectl get all
You can also validate that the app is functional.

curl localhost:4503
Be sure to kill the skaffold process before continuing.

<ctrl+c>

