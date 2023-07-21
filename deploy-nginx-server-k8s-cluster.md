## Tutorial by Allan Guwatudde

In this tutorial we're going to use porter to install an nginx server to a local bare metal kubernetes cluster compromised of 3 rasberrypis

Let's start by creating a directory for our porter bundle. I'll call mine `bare_metal`. I am using a unix based machine. From your terminal emulator run;

```
$ mkdir bare_metal
$ cd bare_metal
```

From here on, all the commands we run are within `bare_metal`
Next let's confirm we can run kubectl commands against our cluster. Run;

```
$ kubectl get nodes -o wide
```

Your output should look like so;
```log
NAME            STATUS   ROLES                  AGE   VERSION    INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
master-node     Ready    control-plane,master   43d   v1.22.17   ...              <none>        Ubuntu 22.04.2 LTS   5.15.0-1032-raspi   containerd://1.6.12
worker-node-1   Ready    <none>                 43d   v1.22.17   ...              <none>        Ubuntu 22.04.2 LTS   5.15.0-1032-raspi   containerd://1.6.12
worker-node-2   Ready    <none>                 43d   v1.22.17   ...              <none>        Ubuntu 22.04.2 LTS   5.15.0-1033-raspi   containerd://1.6.12
```
Now we have the cluster ready. You might need to confirm that your kubeconfig profile has permissions to create resources on the cluster.

Next we need to install porter. You can find installation instructions here https://getporter.org/install/#install-or-upgrade

Confirm installation. Run;

```
$ porter version
```

Your output should look like so;

```log
porter v1.0.14 (0e739d88)
```

Next we bootstrap a bundle. Run;

```
$ porter create
```
 This command will create all the necessary starter files to create your bundle. New directory will look like so;

```log
.
├── .dockerignore
├── .gitignore
├── README.md
├── helpers.sh
├── porter.yaml
└── template.Dockerfile

1 directory, 6 files
``` 

For this tutorial. Let's talk about `helpers.sh` and `porter.yaml`. The `porter.yaml` file is the configuration file that porter uses to do several actions on your bundle during development, installation through to distribution.
It guides porter on how you want it to run some of the steps. For example, what to do during the installation of your bundle. `helpers.sh` file is used by the exec `mixin` to run the commands specified within that file. You can change it's name
if you want, just remember to update the entry in the `porter.yaml` file where it's referenced.

I have mentioned the word `mixin`. What's that in the context of porter? The way porter is able to create a bundle, that you can use to deploy your application is by gluing together a bunch of tools that you would normally use to achieve deployments.
For example, you might have some bash scripts, that create a zip file for all your application files, and then inside that bash script you could call another command to push this file to your servers. You can then run more commands from the terminal using some cli tools e.g. kubectl . Porter's goal is to glue all these tools together into a bundle, together with your application files and then create an OCI artifact that can even be distributed for other people to deploy. 

We have so far mentioned using some bash scripts and cli tools. Now if there's even more tools like terraform to provision your infra, helm etc. How do we get those into the porter bundle build process? We can use mixins. That means there's an exec mixin to run your bash scripts, kubernetes mixin to run kubectl commands, terraform mixin etc. See mixins here https://getporter.org/introduction/intro-mixins/#available-mixins

Let's do some edits to the default `porter.yaml` and customize it for our bundle. We are going to change the bundle name, declare the kubernetes mixin and also add some credentials that will hold the kubeconfig file used by the kubernetes mixin. The Kubernetes mixin internally uses `kubectl` and will therefore need a kubeconfig file passed to it. This is why we shall declare a credential in the `porter.yaml` file.

For this bundle these are the changes made to the porter.yaml

```log
diff --git a/porter.yaml b/porter.yaml
index a4202a1..68178aa 100644
--- a/porter.yaml
+++ b/porter.yaml
@@ -11,16 +11,16 @@ schemaType: Bundle
 schemaVersion: 1.0.1
 
 # Name of the bundle
-name: porter-hello
+name: nginx-bundle
 
 # Version of the bundle. Change this each time you modify a published bundle.
 version: 0.1.0
 
 # Description of the bundle and what it does.
-description: "An example Porter configuration"
+description: "This is a porter configuration used to create nginx server bundle that can be deployed on a kubernetes cluster"
 
 # Registry where the bundle is published to by default
-registry: "localhost:5000"
+registry: "hub.docker.com"
 
 # If you want to customize the Dockerfile in use, uncomment the line below and update the referenced file. 
 # See https://getporter.org/bundle/custom-dockerfile/
@@ -29,19 +29,26 @@ registry: "localhost:5000"
 # Declare and optionally configure the mixins used by the bundle
 mixins:
   - exec
+  - kubernetes
 
 # Define the steps that should execute when the bundle is installed
 install:
   - exec:
-      description: "Install Hello World"
+      description: "Installation steps/commands to run when 'porter install' is used"
       command: ./helpers.sh
       arguments:
         - install
 
+  - kubernetes:
+      description: "Uses kubectl in the background to install k8s components into your cluster"
+      manifests:
+        - /cnab/app/manifests/nginx
+      wait: true
+
 # Define the steps that should execute when the bundle is upgraded
 upgrade:
   - exec:
-      description: "World 2.0"
+      description: "Upgrade commands to run when 'porter upgrade' is used"
       command: ./helpers.sh
       arguments:
         - upgrade
@@ -49,18 +56,17 @@ upgrade:
 # Define the steps that should execute when the bundle is uninstalled
 uninstall:
   - exec:
-      description: "Uninstall Hello World"
+      description: "Uninstall commands"
       command: ./helpers.sh
       arguments:
         - uninstall
 
 # Below is an example of how to define credentials
 # See https://getporter.org/author-bundles/#credentials
-#credentials:
-#  - name: kubeconfig
-#    path: /home/nonroot/.kube/config
-#  - name: username
-#    env: USERNAME
+credentials:
+  - name: kubeconfig
+    path: /home/nonroot/.kube/config
+
```

We have customized our configuration file and made some changes. Since this bundle is intended to deploy nginx server to k8s cluster, we have declared the kubernetes mixin and kubeconfig credential. Under the install declaration, we have told porter to use kubernetes mixin and provided the path where to find the kubernetes manifests. We shall be adding those later to our directory. Furthermore, under the credentials declaration, we have declared  a credential name `kubeconfig` and this will hold the path to the config file required by kubectl.

Now with the configuration ready. Let's use it to guide us to do the remaining steps required to get our bundle ready. 
First we need to install the porter kubernetes mixin. Run;
```
$ porter mixin install kubernetes
```

Next we need to create a credentials file that will hold our path to kubeconfig on our machine. Note that if we were pulling a bundle from an OCI repo we would simple use `$ porter credentials generate`. Run;
```
$ porter credentials create nginx-cred-set.yaml
```

This creates a credentials file called `nginx-cred-set.yaml` and in there we can pass the path to our kubeconfig file.
```log
.
├── .dockerignore
├── .gitignore
├── README.md
├── helpers.sh
├── nginx-cred-set.yaml
├── porter.yaml
└── template.Dockerfile

1 directory, 7 files
```

 Edit it and it should look like so;
```yaml
schemaType: CredentialSet
schemaVersion: 1.0.1
name: k8s-cluster-creds
credentials:
  - name: kubeconfig
    source:
      path: /Users/allan/Desktop/cluster_credentials/config

```

Next we need to create our kubernetes manifest files. We're going to create a directory called manifests and in there we shall place `nginx-deployment.yaml` to create nginx pods on the cluster as well as `nginx-service.yaml` to create a service that can allow us access our nginx server from outside the cluster.

```log
.
├── README.md
├── helpers.sh
├── manifests
│   └── nginx
│       ├── nginx-deployment.yaml
│       └── nginx-service.yaml
├── nginx-cred-set.yaml
├── porter.yaml
└── template.Dockerfile

3 directories, 7 files
```

Your k8s manifest files could look different, but mine look like so;

nginx-deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: porter-bundle-nginx-deployment
  labels:
    app: porter-bundle-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: porter-bundle-nginx
  template:
    metadata:
      labels:
        app: porter-bundle-nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
        ports:
        - containerPort: 80
```

nginx-service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: porter-bundle-nginx-service
  labels:
    app: porter-bundle-nginx
spec:
  type: NodePort
  selector:
    app: porter-bundle-nginx
  ports:
    - protocol: TCP
      port: 8999
      targetPort: 80
      nodePort: 30002

```

All the files required are now ready. It's time to run the porter commands that actually create our bundle.

Let's begin by building the bundle. Run;
```
$ porter build
```

This adds a new directory called `.cnab` that holds the bundle files. See

```log
.
├── .cnab
│   ├── Dockerfile
│   ├── app
│   │   ├── mixins
│   │   │   ├── exec
│   │   │   │   ├── exec
│   │   │   │   └── runtimes
│   │   │   │       └── exec-runtime
│   │   │   └── kubernetes
│   │   │       ├── kubernetes
│   │   │       └── runtimes
│   │   │           └── kubernetes-runtime
│   │   ├── porter.yaml
│   │   ├── run
│   │   └── runtimes
│   │       └── porter-runtime
│   └── bundle.json
├── .dockerignore
├── .gitignore
├── README.md
├── helpers.sh
├── manifests
│   └── nginx
│       ├── .cnab
│       │   └── app
│       ├── nginx-deployment.yaml
│       └── nginx-service.yaml
├── nginx-cred-set.yaml
├── porter.yaml
└── template.Dockerfile

13 directories, 18 files
```

Now the bundle is ready! At this point we can now distribute it by running `porter publish`. This command is similar to `docker push` and would push the OCI bundle to your configured repository.


Let's use our created bundle. We need to apply our credentials to the newly created bundle. Run;

```
$ porter credentials apply nginx-cred-set.yaml
```

If we now run, `$ porter credentials list`, we should see;

```log
------------------------------------------------
  NAMESPACE  NAME               MODIFIED        
------------------------------------------------
             cluster-creds      10 seconds ago
```

Now the bundle is ready! At this point we can now distribute it by running `porter publish`. This command is similar to `docker push` and would push the OCI bundle to your configured repository. 

But we shall not be publishing this bundle just yet. Instead we want to use it to deploy nginx to k8s cluster. This is very easy. Run;

```
$ porter install -c cluster-creds
```

This is my output on stdout;
```log
executing install action from nginx-bundle (installation: /nginx-bundle)
Installation steps/commands to run when 'porter install' is used
Hello World
Uses kubectl in the background to install k8s components into your cluster
deployment.apps/porter-bundle-nginx-deployment created
service/porter-bundle-nginx-service created
execution completed successfully!
```

As you can see the installation was successful. But let's verify this from the cluster. Run;

```
$ kubectl --kubeconfig /Users/allan/Desktop/cluster_credentials/config get pods -l app=porter-bundle-nginx 
```

The output confirms successful deployment;

```log
NAME                                             READY   STATUS    RESTARTS   AGE
porter-bundle-nginx-deployment-58ccd9564-fmzvb   1/1     Running   0          13m
porter-bundle-nginx-deployment-58ccd9564-hxbtb   1/1     Running   0          13m
```

```
$ kubectl --kubeconfig /Users/allan/Desktop/cluster_credentials/config get svc -l app=porter-bundle-nginx
```

```log
NAME                          TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
porter-bundle-nginx-service   NodePort   10.107.172.57   <none>        8999:30002/TCP   16m
```
