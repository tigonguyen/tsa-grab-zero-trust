# tsa-grab-zero-trust
This is belong to Tiny-Scale Architecture blog series.

## Todolist
- [x] Create vault and its PKI engine
- [x] Create Kafka Cluster with Strimzi
- [x] Implement Cluster CA 
- [ ] Implement Client CA
- [ ] OPA Authorization


## Introduction
The purpose of this lab is to create a 'tiny-scale' architecture that simulates the Zero Trust system used at Grab. Zero Trust is a security model that assumes no part of the network is secure and requires verification for every access request. This model is crucial for protecting sensitive data and ensuring secure communication between services.

Although I haven't been part of the Grab team that set up this architecture, I have thoroughly researched their approach by studying their blog posts [here](https://engineering.grab.com/zero-trust-with-kafka). Combining this information with my own knowledge and experience, I aim to implement a similar basic workflow. This will help us understand the fundamental concepts and practices of Zero Trust security in a Kubernetes environment.

Grab uses a production-ready Kubernetes cluster to run this architecture. For this lab, I'll use minikube as our Kubernetes environment. Minikube is an open-source tool that lets us run a single-node Kubernetes cluster locally, providing a powerful yet simplified setup perfect for development and testing. It includes many features of a full-scale Kubernetes setup but reduces the complexity, making it easier to manage and understand. Using minikube, we can create an environment that mimics a production Kubernetes cluster while remaining easy to set up and work with. You can find the entire architecture diagram here:

## Grab's Architecture Concept
Grab’s real-time data platform team, also known as Coban, decided to use mutual Transport Layer Security (mTLS) for authentication and encryption. mTLS enables clients to authenticate servers, and servers to reciprocally authenticate clients.

They opted for Hashicorp Vault and its PKI engine to dynamically generate clients and servers’ certificates. This enables them to enforce the usage of short-lived certificates for clients, which is a way to mitigate the potential impact of a client certificate being compromised or maliciously shared.

For authorisation, they chose Policy-Based Access Control (PBAC), a more scalable solution than Role-Based Access Control (RBAC), and the Open Policy Agent (OPA) as their policy engine, for its wide community support.

To integrate mTLS and the OPA with Kafka, the Coban leveraged Strimzi, the Kafka on Kubernetes operator. They have alluded to Strimzi and hinted at how it would help with scalability and cloud agnosticism. Built-in security is undoubtedly an additional driver of their adoption of Strimzi.
