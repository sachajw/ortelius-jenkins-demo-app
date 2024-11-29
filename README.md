Hello World sample shows how to deploy [SpringBoot](http://projects.spring.io/spring-boot/) RESTful web service application with [Docker](https://www.docker.com/) and with [Kubernetes](https://kubernetes.io/)

#### Prerequisite

Installed:
[Docker](https://www.docker.com/)
[git](https://www.digitalocean.com/community/tutorials/how-to-contribute-to-open-source-getting-started-with-git)

Optional:
[Docker-Compose](https://docs.docker.com/compose/install/)
[Java 1.8 or 11.1](https://www.oracle.com/technetwork/java/javase/overview/index.html)
[Maven 3.x](https://maven.apache.org/install.html)

#### Steps

##### Clone source code from git

```shell
git clone https://github.com/dstar55/docker-hello-world-spring-boot .
```

##### Build Docker image

```shell
docker build -t="hello-world-java" .
```
Maven build will be executes during creation of the docker image.

>Note:if you run this command for first time it will take some time in order to download base image from [DockerHub](https://hub.docker.com/)

##### Run Docker Container

```shell
docker run -p 8080:8080 -it --rm hello-world-java
```

##### Test application

```shell
curl localhost:8080
```

response should be:

```shell
Hello World
```

#####  Stop Docker Container:

```shell
docker stop `docker container ls | grep "hello-world-java:*" | awk '{ print $1 }'`
```

### Run with docker-compose

Build and start the container by running

```shell
docker-compose up -d
```

#### Test application with ***curl*** command

```shell
curl localhost:8080
```

response should be:

```shell
Hello World
```

##### Stop Docker Container:

```shell
docker-compose down
```

### Deploy under the Kuberenetes cluster

#### Prerequisite

##### MiniKube

Installed:
[MiniKube](https://www.digitalocean.com/community/tutorials/how-to-use-minikube-for-local-kubernetes-development-and-testing)

Start minikube with command:

```shell
minikube start
```

#### Retrieve and deploy application

```shell
kubectl create deployment hello-spring-boot --image=dstar55/docker-hello-world-spring-boot:latest
```

#### Expose deployment as a Kubernetes Service

```shell
kubectl expose deployment hello-spring-boot --type=NodePort --port=8080
```

#### Check whether the service is running

```shell
kubectl get service hello-spring-boot
```

response should something like:

```shell
NAME                TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
hello-spring-boot   NodePort   xx.xx.xxx.xxx   <none>        8080:xxxxx/TCP   59m
```

#### Retrieve URL for application(hello-spring-boot)

```shell
minikube service hello-spring-boot --url
```

response will be http..., e.g:

```shell
http://127.0.0.1:44963
```

#### Test application with ***curl*** command(note: port is randomly created)

```shell
curl 127.0.0.1:44963
```

response should be:

```shell
Hello World
```
