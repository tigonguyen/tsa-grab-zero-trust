# tsa-grab-zero-trust
This is belong to Tiny-Scale Architecture blog series.

## Todolist
- [ ] Create vault and its PKI engine
- [ ] Create Kafka Cluster with Strimzi
- [ ] Implement Server Authentication
- [ ] Implement Client Authentication
- [ ] OPA Authorization

## Notes
- Setup for just one environment

## Introduction
The purpose of this lab is to create a 'tiny-scale' architecture that simulates the Zero Trust system used at Grab. Zero Trust is a security model that assumes no part of the network is secure and requires verification for every access request. This model is crucial for protecting sensitive data and ensuring secure communication between services.

Although I haven't been part of the Grab team that set up this architecture, I have thoroughly researched their approach by studying their blog posts [here](https://engineering.grab.com/zero-trust-with-kafka). Combining this information with my own knowledge and experience, I aim to implement a similar basic workflow. This will help us understand the fundamental concepts and practices of Zero Trust security in a Kubernetes environment.

Grab uses a production-ready Kubernetes cluster to run this architecture. For this lab, I'll use minikube as our Kubernetes environment. Minikube is an open-source tool that lets us run a single-node Kubernetes cluster locally, providing a powerful yet simplified setup perfect for development and testing. It includes many features of a full-scale Kubernetes setup but reduces the complexity, making it easier to manage and understand. Using minikube, we can create an environment that mimics a production Kubernetes cluster while remaining easy to set up and work with. You can find the entire architecture diagram here:

## Pre-requisites
- minikube installed
- helm installed
- vault installed, run as client to communicate with vault cluster on minikube
## Configurations steps
### Install and configure Vault
1. Vault installation:
In this section, we will set up a basic Vault cluster running on `minikube`. In the scope of this lab, we'll keep it on the same Kubernetes cluster but run it in a separate namespace to simulate a dedicated Vault cluster, similar to what Grab uses.

First of all, we need to start our Minikube, which will be the main environment used for this lab. And then create a namespace dedicated for Vault cluster.
```
minikube start
kubectl create namespace vault
alias kv='kubectl -n vault' # make an alias for future queries
```

Add the HashiCorp Helm repository to your local Helm, then install Vault chart with the default values:
```
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
helm install vault hashicorp/vault -n vault
```

Wait until Vault is up and running. You should see output similar to this when executing the `kv get pod,svc`
```
NAME                                       READY   STATUS    RESTARTS   AGE
pod/vault-0                                1/1     Running   0          40m
pod/vault-agent-injector-d986fcb9b-ckrkv   1/1     Running   0          40m

NAME                               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/vault                      ClusterIP   10.99.87.187    <none>        8200/TCP,8201/TCP   40m
service/vault-agent-injector-svc   ClusterIP   10.100.251.30   <none>        443/TCP             40m
service/vault-internal             ClusterIP   None            <none>        8200/TCP,8201/TCP   40m
```