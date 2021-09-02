# space-1

17: Skaffold
Skaffold

Thus far, to deploy any changes to the app, we have needed to re-build the image, push it to the docker registry, recreate the pod in Kubernetes, and potentially port-forward for testing. We can automate these steps (and more) using a command line tool called Skaffold.

Skaffold facilitates continuous development for Kubernetes applications by handling the workflow for building, pushing and deploying your application.
It simplifies the development process by combining multiple steps into one easy command
It provides the building blocks for a CI/CD process
Confirm Skaffold is installed by running the following command

skaffold version
Adding Skaffold YAML

Skaffold is configured using…you guessed it…another YAML file
Create a YAML file skaffold.yaml in the root of the project
apiVersion: skaffold/v2beta12
kind: Config
metadata:
  name: k-s-demo-app--
build:
  artifacts:
  - image: eduk8s-labs-w02-s204-registry.springone2021-49e5522.tanzu-labs.esp.vmware.com/apps/demo
    buildpacks:
      builder: docker.io/paketobuildpacks/builder:base
      dependencies:
        paths:
        - src
        - pom.xml
deploy:
  kubectl:
    manifests:
    - k8s/deployment.yaml
    - k8s/service.yaml
    - k8s/ingress.yaml
portForward:
- resourceType: service
  resourceName: k8s-demo-app 
  port: 80
  localPort: 4503
The builder is the same one used by Spring Boot when it builds a container from the build plugins (you would see it logged on the console when you build the image).

An alternative syntax for the build section would be to use custom instead of buildpacks configuration. For example:

   custom:
      buildCommand: ./mvnw spring-boot:build-image -D spring-boot.build-image.imageName=$IMAGE && docker push $IMAGE
      
  18: Development with Skaffold
Skaffold makes some enhancements to our development workflow when using Kubernetes

Build the app and create the container (buildpacks)
Push the container to the registry (Docker)
Apply the deployment and service YAMLs
Stream the logs from the Pod to your terminal
Automatically set up port forwarding
Run the following command to have Skaffold build and deploy our application to Kubernetes.

skaffold dev --port-forward
Testing Everything Out

If you use watch to view your Kubernetes resources you will see the same resources created as before

watch -n 1 kubectl get all
After the resources are created you can stop the watch command.

<ctrl+c>
When running skaffold dev --port-forward you will see a line in your console that looks like this

Port forwarding service/k8s-demo-app in namespace eduk8s-labs-w01-s010, remote port 80 -> address 127.0.0.1 port 4503
In this case port 4503 will be forwarded to port 80 of the service

NOTE: Your port may be different from 4503 please verify with the output from skaffold dev --port-forward and substitute your correct port if needed.

curl localhost:4503

19: Make Changes to the Controller
Skaffold will watch the project files for changes. Open K8sDemoApplication.java and change the hello method to return Hola World.

return
Once you save the file, return to the Terminal and you will notice Skaffold rebuild and redeploy everything with the new change

When you see the port forwarding message again, execute the following command and notice it now returns Hola World.

curl localhost:4503

20: Cleaning Everything Up
Once we are done, if we kill the skaffold process, Skaffold will remove all the resources. Just hit CTRL-C in the terminal where skaffold is running…

To exit Skaffold in terminal one.

<ctrl+c>
You should see the following output.

...
WARN[2086] exit status 1
Cleaning up...
 - deployment.apps "k8s-demo-app" deleted
 - service "k8s-demo-app" deleted
 - ingress.networking.k8s.io "k8s-demo-app" deleted

21: Debugging with Skaffold
Skaffold also makes it easy to attach a debugger to the container running in Kubernetes

skaffold debug --port-forward 
...
Port forwarding service/k8s-demo-app in namespace rbaxter, remote port 80 -> address 127.0.0.1 port 4503
Watching for changes...
Port forwarding pod/k8s-demo-app-75d4f4b664-2jqvx in namespace rbaxter, remote port 5005 -> address 127.0.0.1 port 5005
...
The debug command results in two ports being forwarded

The http port, 4503 in the above example
The remote debug port 5005 in the above example
Set a breakpoint where we return Hola World from the hello method in K8sDemoApplication.java

return
Example: To add a breakpoint, click to the left of the line number in the Editor. Notice the dot on the return statement (Click image to enlarge).alt_text

22: Create a Launcher
To debug our application running in Kubernetes we need to create a remote debug configuration in our IDE. This will work for any remote process (doesn’t have to be running in Kubernetes).

On the left hand side of the IDE tab, click the Run/Debug icon to open the Run/Debug panel, then click on the create a launch.json file link.

alt_text

The IDE will create a default launch configuration for the current file and for K8sDemoAppApplication. We need to add another configuration for remote debugging.

Copy and paste the following JSON snippet into your launch.json ABOVE the first json entry (inside the configurations block).

{
    "type": "java",
    "name": "Debug (Attach)",
    "request": "attach",
    "hostName": "localhost",
    "port": 5005
},
Now select the Debug (Attached) option from the drop down and click the Run button

This should attach the debugger to the remote port (5005) established by Skaffold

alt_text

Now you can execute the following command to make a request to the application. You will not see a response immediately because the debugger in the IDE will break at our breakpoint. Return to the Editor and notice the debug toolbar in the top center of the screen, and the debug info frames on the left and bottom of the screen. You can step through the code, view variable values, etc.

curl localhost:4503
Stop the Skaffold process by executing the following command

<ctrl+c>

