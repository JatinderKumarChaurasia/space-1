# space-1

34: Removing The Config Map and Reverting The Deployment
Before continuing lets remove our config map and revert the changes we made to deployment.yaml To delete the ConfigMap run the following command
kubectl delete configmap log-level
In kustomize/base/deployment.yaml remove the envFrom properties we added

sed -i '39,41d' ~/demo/kustomize/base/deployment.yaml
Next we will use Kustomize to make generating ConfigMaps easier
Your deployment.yaml should be like this:

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
        name: demo
        resources: {}
        readinessProbe:
          httpGet:
            port: 8080
            path: /actuator/health/readiness
        livenessProbe:
          httpGet:
            port: 8080
            path: /actuator/health/liveness
        lifecycle:
          preStop:
            exec:
              command:
                - sh
                - '-c'
                - sleep 10
status: {}


35: Config Maps and Spring Boot Application Configuration
In Spring Boot we usually place our configuration values in application properties or YAML files
ConfigMaps in Kubernetes can be populated with values from files, like properties or YAML files
We can do this via kubectl
kubectl create configmap k8s-demo-app-config --from-file ./path/to/application.yaml
No need to execute the above command, it is just an example, the following sections will show a better way.

We can then mount this config map as a volume in our container at a directory Spring Boot knows about and Spring Boot will automatically recognize the file and use it

36: Creating A Config Map With Kustomize
Kustomize offers a way of generating ConfigMaps and Secrets as part of our customizations.

Create a file called application.yaml in kustomize/base and add the following content

logging:
  level:
    org:
      springframework: INFO
We can now tell Kustomize to generate a ConfigMap from this file, in kustomize/base/kustomization.yaml by adding the following snippet to the end of the file
configMapGenerator:
  - name: k8s-demo-app-config
    files:
      - application.yaml
If you now run $ kustomize build you will see a ConfigMap resource is produced
kustomize build kustomize/base
apiVersion: v1
data:
  application.yaml: |-
    logging:
      level:
        org:
          springframework: INFO
kind: ConfigMap
metadata:
  name: k8s-demo-app-config-fcc4c2fmcd
By default kustomize generates a random name suffix for the ConfigMap. Kustomize will take care of reconciling this when the ConfigMap is referenced in other places (ie in volumes). It does this to force a change to the Deployment and in turn force the app to be restarted by Kubernetes. This isn’t always what you want, for instance if the ConfigMap and the Deployment are not in the same Kustomization. If you want to omit the random suffix, you can set behavior=merge (or replace) in the configMapGenerator.

Now edit deployment.yaml in kustomize/base to have Kubernetes create a volume for the ConfigMap and mount that volume in the container

Create the volume.

volumes:
  - name: config-volume
    configMap:
      name: k8s-demo-app-config
Mount the volume.

volumeMounts:
  - name: config-volume
    mountPath: /workspace/config
In the above deployment.yaml we are creating a volume named config-volume from the ConfigMap named k8s-demo-app-config
In the container we are mounting the volume named config-volume within the container at the path /workspace/config
Spring Boot automatically looks in ./config for application configuration and if present will use it (because the app is running in /workspace)
Your deployment.yaml

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
      - image: lab-workshop-1-w01-s001-registry.192.168.64.7.nip.io/apps/demo
        name: demo
        resources: {}
        readinessProbe:
          httpGet:
            port: 8080
            path: /actuator/health/readiness
        livenessProbe:
          httpGet:
            port: 8080
            path: /actuator/health/liveness
        lifecycle:
          preStop:
            exec:
              command:
                - sh
                - '-c'
                - sleep 10
        volumeMounts:
          - name: config-volume
            mountPath: /workspace/config
      volumes:
        - name: config-volume
          configMap:
            name: k8s-demo-app-config
status: {}
Your base/kustomization.yaml

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:    
- service.yaml
- deployment.yaml
- ingress.yaml
configMapGenerator:
  - name: k8s-demo-app-config
    files:
      - application.yaml
Your base/application.yaml

logging:
  level:
    org:
      springframework: INFO
      
  
 
 37: Testing Our New Deployment
If you run $ skaffold dev --port-forward everything should deploy as normal

skaffold dev --port-forward
Check that the ConfigMap was generated

kubectl get configmap
You should see something similar to this

NAME                             DATA   AGE
k8s-demo-app-config-fcc4c2fmcd   1      18s
Skaffold is watching our files for changes, go ahead and change logging.level.org.springframework from INFO to DEBUG and Skaffold will automatically create a new ConfigMap and restart the pod

This is the field that you will be changing

springframework
To change INFO to DEBUG

sed -i s/INFO/DEBUG/g ~/demo/kustomize/base/application.yaml
You should see a lot more logging in your terminal once the new pod starts

Be sure to kill the skaffold process before continuing

<ctrl+c>
Also, you will want to go back to application.properties in kustomize/base and change logging.level.org.springframework back to INFO.

sed -i s/DEBUG/INFO/g ~/demo/kustomize/base/application.yaml
To verify springframework is INFO again:

springframework

