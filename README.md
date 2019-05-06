# Kubernetes with Meteor. Basics
draft v 1

### Intro
If you are using Meteor for real world projects sooner or later you will face that:

- Your app doesn't handle well the increasing the load. Meteor and primarily Node.js doesn't support multi-core parallelism so there is little use of vertical scaling. For horizontal scaling you need to split your users between separate instances of the identical app.
- Deploying new versions becomes more of a struggle.
- Your server becomes complicated and it will be a difficult task to create a copy of it for a similar project or for a horizontal scaling.
- Some parts of your monolithic project should be divided into smaller pieces (services) perhaps even rewritten using other technological stack.
- You need to operate everything by hands.

Kubernetes is a great tool in dealing with all of these things and more.

This guide will give a basic knowledge of how to deploy a Meteor app with Kubernetes. 

### Kubernetes. Brief info
Kubernetes is the well know tool, eco-system and platform created by Google ([https://kubernetes.io/](https://kubernetes.io/)). 
Its open-source and you can run it anywhere you want or use existing cloud services
like [Google Cloud](https://cloud.google.com/kubernetes-engine/).

Kubernetes is used for deploying and orchestrating applications. In heart of it lies a containerized applications mainly [docker containers](https://en.wikipedia.org/wiki/Docker_(software). 

Containers killing feature is the ability to encapsulate your application and the required environment (OS, other applications, system settings, etc) into a relatively small isolated chunks called *images*. 
Think of it as an *operating-system-level virtualization*.

Kubernetes operates the containers and allows you to use different stack of technologies
for your project (Meteor, plain Node.js, Java, Python and everything else).

Kubernetes consists of tools and a cluster. 

The cluster is where your apps are going to run. The cluster works on a number of physical or cloud machines bound with network. 
These machines are called _Nodes_.

The tools are used to manipulate the cluster and everything inside of it. 
Most of the time you will create YAML files (declarative _instructions_) and pass them (_apply_)
to the cluster.

 Here are the main Kubernetes features mentioned on the official website:
- **Service discovery and load balancing**. Means that you can balance the load (traffic, for example) across your apps, give routes to your apps, etc.
- **Horizontal scaling**. You can create as many active instances of your app (or even use your own script for smart scaling) as you want and your computational power allows.
- **Storage orchestration**. Mount any storage system your app needs (local storage, cloud storage, network storage).
- **Automated rollouts and rollbacks**. Allows you to rollout new versions of your app without any downtime and rollback to previous versions in case of an error.
- **Batch execution**. Kubernetes can manage your batch and CI workloads, replacing containers that fail, if desired.
- **Automatic bin packing**. Optimal usage of all the computational resources.
- **Self-healing**. If a container fails Kubernetes is smart enough to create a new one which replaces the failed.
- **Secret and configuration management**. You can store environment variables, sensitive information, passwords or configurations for your app separately from it. Changes in secrets doesn't require rebuilding the image.

There are a lot of information about Kubernetes on the Internet, if you are not familiar with it you should
start with the basics [https://kubernetes.io/docs/tutorials/kubernetes-basics/](https://kubernetes.io/docs/tutorials/kubernetes-basics/) but **it is not mandatory**.
If you are going to use cloud Kubernetes providers to deploy your Meteor project this guide is enough for
you to start.

### Required Terminology
- **Image**. Your app inside build Docker container.
- **Pod**. Instance of an Image inside Kubernetes Cluster.
- **Replica set**. Controls the number of the identical Pods.
- **Secret**. Encrypted information inside the Kubernetes Cluster.
- **Deployment**. Your app, its version, its settings, Docker Image, required Secrets, Replica policy and  more. 
- **Service**. Groups Pods of your app.
- **Load balancer**. Splits the incoming load between Services and their Pods.

### 1. Creating a Docker Image
First of all you need to install Docker on the machine where you are going to build your Meteor project.
You can read about installation here [https://docs.docker.com/install/](https://docs.docker.com/install/).

You can create a Docker image from scratch but we are going to use `abernix/meteord:onbuild` image as a the base image. This image was specially created for Meteor, uses Ubuntu and builds your Meteor project into Node.js automatically.

You can find more Docker images for Meteor at[ Abernix hub](https://hub.docker.com/r/abernix/meteord).

Make a text file with name `Dockerfile` in the root of your project.

The file contents should be:
```FROM abernix/meteord:onbuild```

The`Dockerfile` can contain other useful references and commands beside the base image. 
[Read more](https://docs.docker.com/engine/reference/builder/).

For example, you can include `graphicsmagick` into your Docker image.
The `Dockerfile` should look like this:
```
FROM abernix/meteord:onbuild
RUN apt-get update && apt-get install graphicsmagick -y
```

### 2. Building an Image, Docker Hub Registry
After you build your Image you have to put it somewhere so Kubernetes Cluster can use the Image to create the Pods. For this purpose exists Docker Registry. You can run your local Docker Registry, run Docker Registry on a dedicated server or even inside the Kubernetes Cluster.

In this guide we are going to use existing solution _Docker Hub_.

Docker Hub is like _npm_ but for containers. 
It has _free_ pricing plan for individuals which allows you to have *unlimited* public repositories, *1* private repository and *1* parallel build (May, 2019).

[Create an account on Docker Hub](https://hub.docker.com/). You will need **username**, **email** and **password**.

Now you can authenticate on your building machine with this command:
```
docker login --username your_username --password your_password
```

To build a Meteor project run the following command in the terminal from your project's root directory:
```
docker build -t your_username/project_name:some_tag .
```
- `.` means that Dockerfile is in the same directory.
- `-t` means that we want to tag the built Image to distinguish it among different projects and builds.
You can use a projects build version in place of `some_tag` like `1.0.0`.

After the build you should upload the Image to Docker Registry:
```
docker push your_username/project_name
```

### 3. Running a Kubernetes Cluster


### 4.0 What is a Secret
As we can find in the official docs:
> Kubernetes secret objects let you store and manage sensitive information, such as passwords, OAuth tokens, and ssh keys. Putting this information in a secret is safer and more flexible than putting it verbatim in a Pod Lifecycle definition or in a container image .

[Read more about Kubernetes secrets](https://kubernetes.io/docs/concepts/configuration/secret/).

### 4.1 Creating a Secret for Docker Hub
To create a Secret for pulling Images from Docker Registry run:
```
kubectl create secret docker-registry regsec \
--docker-server=https://index.docker.io/v1/ \
--docker-username=your_username \
--docker-password=your_password \
--docker-email=your_email
```

where:
- **regsec** is secret's name
- **docker-server** is your Private Docker Registry. (https://index.docker.io/v1/ for DockerHub)
- **docker-username** is your Docker username.
- **docker-password** is your Docker password.
- **docker-email** is your Docker email.

[Read more](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/).

### 4.2 Creating Secrets for ENVs
Secrets are good for storing environment variables for further use inside the containers.
We going to create secrets both for **METEOR_SETTINGS** and **MONGO_URL** environment variables.

Let's assume you have a file with Meteor settings called _meteor_settings.json_.
```
{
  "public": {},
  "someThingCustom": {
    "first": 0,
    "last": 10
  }
}
```
To create a Secret from this file run:
```kubectl create secret generic meteor-settings --from-file=./meteor_settings.json```

This will create a Secret with name `my-meteor-settings` which includes only one key `meteor_settings.json`.

The same way you can create a _mongo_url.txt_ with the desired url and use the same command to create the Secret.

Create a file _mongo_url.txt:
```
mongodb://login:password@ip:port/db
```

Replace **login**, **password**, **ip**, **port** and **db** with your data.

Run:
```
kubectl create secret generic mongo-url --from-file=./mongo_url.txt
```

You will see how to use created ENVs in deployment section of this guide.

### 4.3 Updating a Secret
If you need to update a Secret the easiest way is to delete the Secret and create it again.

To delete a Secret run:
```
kubectl delete secret secret_name
```

If you want to update a Secret created from a file you can use this command:
```
kubectl create secret generic secret_name \
    --from-file=./file.name --dry-run -o yaml | 
  		kubectl apply -f -
```

Its the same command you use to create a Secret plus you add `--dry-run -o yaml | 
  		kubectl apply -f -` in the end.

### 5. Little about the instructions
As it was mentioned everything you pass to Kubernetes is a YAML instruction (even if you write
this instruction in JSON or use commands from terminal it is compiled to YAML). 

So if you need to apply the instruction to Kubernetes from a file or update a
previously applied instruction you can use one simple command for both tasks:
```
kubectl apply -f your_file
```

I suggest to create a special folder for yml files and don't mess them with your project sources.

Read more about [YAML](https://en.wikipedia.org/wiki/YAML).
Read more about [different API versions of YAML instructions](https://matthewpalmer.net/kubernetes-app-developer/articles/kubernetes-apiversion-definition-guide.html).

### 6.1 Deploying a Meteor app
To deploy your app you should create a Deployment.

Create the following YAML file (_deployment.yml_). Everything you need is commented:
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: my-project # your project name
  name: my-project-deployment # your project deployment name
spec:
  progressDeadlineSeconds: 600
  replicas: 1 # number of Pods your deployment creates
  revisionHistoryLimit: 10 # number of old ReplicaSets to retain to allow rollback
  selector:
    matchLabels:
      app: my-project # your project name
  strategy:
    rollingUpdate:
      maxSurge: 25% # maximum number of Pods that can be created over the desired number of Pods
      maxUnavailable: 25% # maximum number of Pods that can be unavailable during the update process
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: my-project # your project name
    spec:
      containers:
      - env: # ENV variables for your project
        - name: ROOT_URL
          value: https://my.domain/ # string value
        - name: MONGO_URL
          valueFrom: # value from previosly created Secret
            secretKeyRef:
              key: mongo_url.txt
              name: mongo-url
        - name: METEOR_SETTINGS
          valueFrom: # value from previosly created Secret
            secretKeyRef:
              key: meteor_settings.json
              name: meteor-settings
        image: your_username/project_name:some_tag # image of your app
        imagePullPolicy: Always
        name: my-project # your project name
        ports:
        - containerPort: 80 # which port should be open on Pods
          name: http-server
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /userimages
          name: images
        - mountPath: /userdocuments
          name: documents
      dnsPolicy: ClusterFirst
      imagePullSecrets:
      - name: regsec # secret with Docker Hub credentials
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - emptyDir: {}
        name: images
      - emptyDir: {}
        name: documents
status: {}
```

In this example we assume that the app uses folders `/userimages` and `/userdocuments` for temp files.

For this we describe `volumes` type of `emptyDir` (you can check other types of `volumes` [here](https://kubernetes.io/docs/concepts/storage/volumes/#local)):
```
volumes:
- emptyDir: {}
  name: images
- emptyDir: {}
  name: documents
```

And mount them with corresponding dir names:
```
volumeMounts:
- mountPath: /userimages
  name: images
- mountPath: /userdocuments
  name: documents
```

To create a Deployment run:
```
kubectl apply -f deployment.yml
```

[Read more](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

### 6.2 Updating the app

To update your app in Kubernetes cluster simply build new image with new tag and push it to Docker Hub.

Then update _deployment.yml_ with new image name and tag `image: your_username/project_name:some_tag`.

Run to apply changes:
```
kubectl apply -f deployment.yml
```

### 7. Auto scaling
Kubernetes allows you to auto scale your deployment based on CPU usage.

For this create the following YAML file (_autoscaler.yml_):
```
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: my-project
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: extensions/v1beta1
    kind: Deployment
    name: my-project
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

where: 
- `minReplicas` is a minimum number of Pods
- `maxReplicas` is a maximum number of Pods
- `targetCPUUtilizationPercentage` is a CPU utilization limit in percents when a new Pod should be created.

Don't forget to replace `my-project` with the name of deployed app.

To apply this YAML instruction run:
```
kubectl apply -f autoscaler.yml
```

[Read more](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/).

### 8. Creating a Service
Your app is deployed and functioning but it is invisible to outside world. 

To make it visible first of all you should create a corresponding Service for your app.
> Service  is an abstraction which defines a logical set of Pods and a policy by which to access them.

[Read more.](https://kubernetes.io/docs/concepts/services-networking/service/)

Create the following YAML file (_service.yml_):
```
kind: Service
apiVersion: v1
metadata:
  name: my-project-service
spec:
  ports:
  - port: 80
    targetPort: http-server
  selector:
    app: my-project
```

This service provides _80_ port on your pods as _http-server_.

Don't forget to replace `my-project` with the name of deployed app.

To apply this YAML instruction run:
```
kubectl apply -f service.yml
```

### 9. Creating an Ingress Controller. Load balancing
Now you app Pods are targeted by a Service. 
Lets make the Service accessible from outside by creating a corresponding Ingress.

> Ingress is an API object that manages external access to the services in a cluster, typically HTTP.
Ingress can provide load balancing, SSL termination and name-based virtual hosting.

[Read more.](https://kubernetes.io/docs/concepts/services-networking/ingress/)

As it was mentioned Ingress also provides load balancing between Service's Pods and SSL/TLS (HTTPS). 
You can achieve this by creating Ingress Controller. 
There are a lot of different controllers, you can find more [here](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/). 

In this guide we use _nginx_ based controller which is officially supported by Kubernetes team.
As you might guess this Ingress Controller will use _nginx_ for load balancing and SSL/TLS termination.

We will use simplest router possible which sends every HTTP request for your domain to one Service.

Create the following YAML file (_ingress.yml_):
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-project-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.org/websocket-services: "my-project"
    nginx.ingress.kubernetes.io/proxy-body-size: 10m
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-hash: "sha1"
spec:
  rules:
  - host: my.domain
    http:
      paths:
      - path: /
        backend:
          serviceName: my-project-service
          servicePort: 80
```

where:
- `nginx.ingress.kubernetes.io/proxy-body-size` is a maximum size of incoming requests,
`10m` means 10 megabytes. For example, if your app allows users to upload their photos,
a user won't be able to upload a file with size more than 10 Mb.

Don't forget to replace `my.domain`, `my-project-service`, `nginx.org/websocket-services`,
`my-project` with your data.

Apply file with:
```
kubectl apply -f ingress.yml
```

Now you have to bind your domain with the load balancer's IP.

To check Ingress Controller IP use command:
```
kubectl get svc -n ingress-nginx | grep ingress | grep LoadBalancer
```

The last step is to delegate DNS _A record_ for your domain and the obtained IP.

### 10.1 HTTPS with custom certificates
If you have custom TLS certificate that you want to use for HTTPS you need to create an corresponding Secret
with `kubectl`:
```
kubectl create secret tls test-secret-tls --cert=server.crt --key=server.key
```

where:
- **test-secret-tls** is the name of the Secret
- **server.crt** is the certificate with public key
- **server.key** is the private key

Now you can modify your _ingress.yml_:
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.org/websocket-services: "my-project"
    nginx.ingress.kubernetes.io/proxy-body-size: 10m
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-hash: "sha1"
    ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - my.domain
    secretName: ingress-tls-secret
  rules:
  - host: lk.nobi.services
    http:
      paths:
      - path: /
        backend:
          serviceName: my-project-service
          servicePort: 80
```

As you can see we added `tls` to `spec` and described there the host we want to accociate with the Ingress and the Secret that holds certificate:
```
tls:
- hosts:
  - my.domain
  secretName: ingress-tls-secret
```

We have also added `ingress.kubernetes.io/ssl-redirect: "true"` to annotations section which forces HTTPS usage over HTTP.

### 10.2 HTTPS with Cert-Manager (auto Let's Encrypt certificates)
**WIP**
[https://docs.cert-manager.io/en/latest/tutorials/acme/quick-start/index.html](https://docs.cert-manager.io/en/latest/tutorials/acme/quick-start/index.html)

### Best practices
Usually we do automated testing of our application locally but its also a good idea to check that 
everything works as expected in real production before it is available to the endpoint user.
The suggest solution is to create a test deployment with a separate domain (test.my.domain) and Ingress
and deploy your new project version to test zone first.

Here you can find a guide of using basic auth with your Nginx Ingress [https://imti.co/kubernetes-ingress-basic-auth/](https://imti.co/kubernetes-ingress-basic-auth/).

### Pipelines
**WIP**

### Meteor and micro-services. Programmable scaling
**WIP**

