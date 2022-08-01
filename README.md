# GRPC Go microservice example (Docker & K8s)
This repository is a example of some grpc capabilities in Go, using docker and k8s (for maximum scalability).

## Pre: Install Go 
```
$ wget https://go.dev/dl/go1.18.4.linux-amd64.tar.gz
$ sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.18.4.linux-amd64.tar.gz

$ grep GO ~/.bashrc 
export GOROOT=/usr/local/go
export PATH=${GOROOT}/bin:${PATH}
export GOPATH=$HOME/go
export PATH=${GOPATH}/bin:${PATH}

$ go version
go version go1.18.4 linux/amd64
```

## Compiling the proto files (gRPC):

Install the protocol compiler plugins for Go using the following commands (Ref: https://grpc.io/docs/languages/go/quickstart/):
```
$ go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.28
$ go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.2
```
Fist we need to compile all files that will define our grpc protocol, all coded in the file functions/functions.proto with the following command:

`$ cd functions && protoc --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative functions.proto && cd ..`

Or we can use docker image (for CI/CD pipeline):

```
$ cd functions 
$ docker build -t davarski/protoc-go .
$ docker login
$ docker push davarski/protoc-go

$ docker run -it --rm -v $PWD:/opt/functions -w /opt/functions davarski/protoc-go
root@4eb705f9eea5:/opt/functions# protoc --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative functions.proto
root@4eb705f9eea5:/opt/functions# ls -tal
total 36
-rw-r--r-- 1 root root 6393 Aug  1 08:22 functions.pb.go
-rw-r--r-- 1 root root 8564 Aug  1 08:22 functions_grpc.pb.go
drwxr-xr-x 1 root root 4096 Aug  1 08:22 ..
drwxr-xr-x 2 1000 1000 4096 Aug  1 08:17 .
-rw-r--r-- 1 1000 1000  421 Aug  1 08:07 Dockerfile
-rw-r--r-- 1 1000 1000  413 Aug  1 08:07 functions.proto
root@4eb705f9eea5:/opt/functions# exit
exit

```

## Compiling the project locally
After that, the go files for the client and server can be compiled and run (in separate terminals) locally for testing the code itself and ensure everythig runs correctly:

`go build -o grpc_server ./server/`
`go build -o grpc_client ./client/`

After running both executables (first the sever then the client) it is possible to see one window receiving strings and the other receiving echos of those strings 

```
$ ./grpc_server 
2022/08/01 10:30:37 listening at: [::]:50501

$ ./grpc_client 
2022/08/01 10:31:21 Echoed: Hello 

```

## Running with Docker
The project can be converted to docker images with the commands:

`docker build -f server/Dockerfile -t davarski/grpc-server .`
`docker build -f client/Dockerfile -t davarski/grpc-client .`

Creating a proper local network for both containers to communicate is necessary:

`docker network create grpc`

Running both containers is 

```
$ docker run -d --net grpc --name grpc-server -p 50501:50501 davarski/grpc-server:latest
$ docker run --net grpc davarski/grpc-client:latest
2022/08/01 06:59:30 Permuted: 000
2022/08/01 06:59:30 Permuted: 010101
2022/08/01 06:59:30 Permuted: 021202101
2022/08/01 06:59:30 Concated: 000111222
2022/08/01 06:59:30 Counted: 0 Hello world
2022/08/01 06:59:30 Counted: 1 Hello world
2022/08/01 06:59:30 Counted: 2 Hello world
2022/08/01 06:59:30 Counted: 3 Hello world
2022/08/01 06:59:30 Counted: 4 Hello world
2022/08/01 06:59:30 Echoed: Hello world
2022/08/01 06:59:31 Counted: 0 Hello world
2022/08/01 06:59:31 Counted: 1 Hello world
2022/08/01 06:59:31 Counted: 2 Hello world
2022/08/01 06:59:31 Counted: 3 Hello world
2022/08/01 06:59:31 Counted: 4 Hello world
```
It should be seen the same as in the local case, however, as the server is running in detached mode to access it's logs:

```
$ docker logs grpc-server
2022/08/01 06:59:18 listening at: [::]:50501
2022/08/01 06:59:30 Received Concat
2022/08/01 06:59:30 Received Echo: world
2022/08/01 06:59:30 Received Counter: 5 world
2022/08/01 06:59:30 Received Permute
2022/08/01 06:59:31 Received Counter: 5 world
2022/08/01 06:59:31 Received Echo: world
2022/08/01 06:59:31 Received Concat
2022/08/01 06:59:31 Received Permute
2022/08/01 06:59:32 Received Permute
2022/08/01 06:59:32 Received Counter: 5 world
2022/08/01 06:59:32 Received Concat
2022/08/01 06:59:32 Received Echo: world
2022/08/01 06:59:33 Received Counter: 5 world
2022/08/01 06:59:33 Received Permute
2022/08/01 06:59:33 Received Echo: world
2022/08/01 06:59:33 Received Concat
2022/08/01 06:59:34 Received Echo: world
2022/08/01 06:59:34 Received Permute
2022/08/01 06:59:34 Received Counter: 5 world
2022/08/01 06:59:34 Received Concat

```
Push to Docker Hub 

```
docker login 
docker push davarski/grpc-server
docker push davarski/grpc-client
```

## K8s deploy (KIND):

```
## Install docker, kubectl, etc.

## Instal KIND

$ curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.14.0/kind-linux-amd64 && chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

### Create cluster (CNI=Calico, Enable ingress)

$ kind create cluster --name devops --config cluster-config.yaml

$ kind get kubeconfig --name="devops" > admin.conf
$ export KUBECONFIG=./admin.conf 

$ kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
$ kubectl -n kube-system set env daemonset/calico-node FELIX_IGNORELOOSERPF=true

### Deploy Go microservcie k8s:

$ kubectl apply -f kubernetes/setup/*
$ kubectl apply -f kubernetes/server.yaml
$ kubectl apply -f kubernetes/client.yaml


$ kubectl get all -n grpc-go
NAME                               READY   STATUS    RESTARTS   AGE
pod/grpc-client-57ff4cb55c-hf7kd   1/1     Running   0          4s
pod/grpc-server-66886c6698-d7hx4   1/1     Running   0          34s
pod/grpc-server-66886c6698-m4tg4   1/1     Running   0          34s
pod/grpc-server-66886c6698-vnfh6   1/1     Running   0          34s

NAME                  TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
service/grpc-server   ClusterIP   10.96.95.144   <none>        50501/TCP   34s

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/grpc-client   1/1     1            1           4s
deployment.apps/grpc-server   3/3     3            3           34s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/grpc-client-57ff4cb55c   1         1         1       4s
replicaset.apps/grpc-server-66886c6698   3         3         3       34s

$ kubectl logs pod/grpc-server-66886c6698-d7hx4 -n grpc-go
2022/08/01 07:26:52 listening at: [::]:50501
$ kubectl logs pod/grpc-client-57ff4cb55c-hf7kd -n grpc-go|head
2022/08/01 07:27:24 Permuted: 000
2022/08/01 07:27:24 Permuted: 010101
2022/08/01 07:27:24 Permuted: 021202101
2022/08/01 07:27:24 Counted: 0 Hello world
2022/08/01 07:27:24 Counted: 1 Hello world
2022/08/01 07:27:24 Echoed: Hello world
2022/08/01 07:27:24 Counted: 2 Hello world
2022/08/01 07:27:24 Counted: 3 Hello world
2022/08/01 07:27:24 Counted: 4 Hello world
2022/08/01 07:27:24 Concated: 000111222


## Clean environment

$ kind delete cluster --name=devops
```


