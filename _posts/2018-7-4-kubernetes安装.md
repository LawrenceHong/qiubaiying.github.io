---
layout:     post
title:      How to use kubernetes
subtitle:  
date:       2018-7-4
author:     Leyou
header-img: img/k8s.png
catalog: true
tags:   
---
> How to install kubernetes.

# Install it.

INSTALLING KUBECTL
```
Linux : $ curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
MacOS : $ curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/darwin/amd64/kubectl
Win : curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.8.0/bin/windows/amd64/kubectl.exe

$ chmod +x ./kubectl
```
Add yo your PATH
```
$ sudo mv ./kubectl /usr/local/bin/kubectl
$ sudo mv ./kubectl /usr/local/bin/kubectl
```

# INSTALL MINIKUBE
```
Linux : $ curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.23.0/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
MacOS : $ curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.23.0/minikube-darwin-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
```

# Start a Demo
```
$ minikube start
$ kubectl run hello-minikube --image=gcr.io/google_containers/echoserver:1.4 --port=8080
$ kubectl expose deployment hello-minikube --type=NodePort
$ kubectl get pod
$ curl $(minikube service hello-minikube --url)
$ kubectl delete deployment hello-minikube
$ minikube stop
```
