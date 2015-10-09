# Kubernetes workshop

This workshop guides through Google Kubernetes, Docker and Jenkins CI

## Activity 0: Setup Kubernetes

Setup and teardown scripts has been placed into ```cluster/kube-up.sh``` and ```cluster/kube-down.sh``` directories correspondingly

To start kubernetes just run following command:
```
$ bash cluster/kube-up.sh
$ kubectl config set-cluster http://localhost:8080
```

To check instalation run:
```
$ kubectl get nodes

NAME        LABELS                             STATUS
127.0.0.1   kubernetes.io/hostname=127.0.0.1   Ready
```

Next step is to clone our micro service
```
$ git clone https://github.com/akranga/chucknorris.git
cd chucknorris
```

## Activity 1: Compiling with Docker

ChuckNorris is a Spring boot micro service with one controller. As the first step we will need to compile it

However our environmetn is not ready. We simply (I don't) do not have Java Development Kit of right version. We will use Java docker container to compile our app and produce distribution JAR file

For the first time we will do it manually:

Simply run command:
```
$ docker run --rm -i -t --volume=$(pwd):/app -w /app java:jdk bas
```

This will download Docker container with latest Open JDK (8); mount current directory and let you inside

Now we can run compilation with gradle
```
root@477c94708216:/app$ ./gradlew assemble

:processResources
:classes
:jar
:findMainClass
:startScripts
:distTar
:distZip
:bootRepackage
:assemble

BUILD SUCCESSFUL

Total time: 19.007 secs
```

Now you can exit container and take a look at the compilation result
```
root@477c94708216:/app# exit
$ ls build/libs/
chnorr-0.1.0.jar
```

Let's do the same compilation as oneliner
```
$ docker run --rm -i -t --volume=$(pwd):/app -w /app java:jdk ./gradlew clean assemble

:processResources
:classes
:jar
:findMainClass
:startScripts
:distTar
:distZip
:bootRepackage
:assemble

BUILD SUCCESSFUL

Total time: 17.534 secs
```

How about to compile and run it? This directory has a Dockerfile. Which where we will put our compiled JAR file and create a container image our of it. Once we have an image then we can run it

Run following command:
```
$ docker build -t akranga/chucknorris .

Sending build context to Docker daemon 37.28 MB
Step 0 : FROM java:jdk
 ---> 7547e52aac4b
Step 1 : ENV APP_VERSION 0.1.0
 ---> Running in ac77614ee44a
...
Step 4 : ENTRYPOINT java -jar /app.jar
 ---> Running in 0991ddf114cf
 ---> c5c729af0559
Removing intermediate container 0991ddf114cf
Successfully built c5c729af0559

$ docker run -d -p 8000:8080 --name=chuck akranga/chucknorris
$ curl localhost:8000

Chuck Norris can read all encrypted data, because nothing can hide from Chuck Norris.
```

Now let's tear down our application
```
$ docker ps

CONTAINER ID        IMAGE                                       COMMAND             
865a6151d7ed        akranga/chucknorris                         "java -jar /app.jar"

$ docker stop 865a6151d7ed
$ docker rm 865a6151d7ed
```

## Activity 2: manipulatipns with Kubernetes

Now let's run our Chuck Norris app as part of kubernetes

Before we start, let's do sanity check
```
$ kubectl get nodes
NAME        LABELS                             STATUS
127.0.0.1   kubernetes.io/hostname=127.0.0.1   Ready
```

Now we are ready to start our Chuck:
```
$ kubectl run chuck --image=akranga/chucknorris --port=8080

CONTROLLER   CONTAINER(S)   IMAGE(S)              SELECTOR    REPLICAS
chuck        chuck          akranga/chucknorris   run=chuck   1

$ kubectl get pods
NAME                   READY     STATUS    RESTARTS   AGE
chuck-bbbj5            1/1       Running   0          3m
```

Next step is to expose our Chuck norris to the world
```
$ kubectl expose rc chuck --container-port=8080 --port=8000

NAME      LABELS      SELECTOR    IP(S)     PORT(S)
chuck     run=chuck   run=chuck             8000/TCP

$ kubectl describe service chuck

Name:			chuck
Labels:			run=chuck
Selector:		run=chuck
...
IP:			    10.0.0.93
Port:			<unnamed>	8080/TCP
Endpoints:		172.17.0.14:8080

$ curl 172.17.0.14:8080

Chuck Norris can solve the Towers of Hanoi in one move.
```

Next step is to scale our application to 5 replicas
```
$ kubectl scale --current-replicas=1 --replicas=3 rc chuck

scaled

$ kubectl get pods

NAME                   READY     STATUS    RESTARTS   AGE
chuck-2z97v            1/1       Running   0          30s
chuck-bbbj5            1/1       Running   0          16m
chuck-ow5s0            1/1       Running   0          30s

$ kubectl describe service chuck
Name:			chuck
...
Port:			<unnamed>	8080/TCP
Endpoints:		172.17.0.14:8080,172.17.0.15:8080,172.17.0.16:8080
```