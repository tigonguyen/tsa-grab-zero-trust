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
#### 1. Install Vault:

In this section, we will set up a basic Vault cluster running on `minikube`. In the scope of this lab, we'll keep it on the same Kubernetes cluster but run it in a separate namespace to simulate a dedicated Vault cluster, similar to what Grab uses.

First of all, we need to start our Minikube, which will be the main environment used for this lab. And then create a namespace dedicated for Vault cluster.
```
minikube start
kubectl create namespace vault
alias kv='kubectl -n vault' # make an alias for future queries
```

Add the HashiCorp Helm repository to your local Helm, then install Vault chart with default values:
```
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
helm install vault hashicorp/vault -n vault
```

Wait until Vault is up and running. You should see output similar to this when executing below command:
```
$ kv get pod,svc
NAME                                       READY   STATUS    RESTARTS   AGE
pod/vault-0                                0/1     Running   0          40m
pod/vault-agent-injector-d986fcb9b-ckrkv   1/1     Running   0          40m

NAME                               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/vault                      ClusterIP   10.99.87.187    <none>        8200/TCP,8201/TCP   40m
service/vault-agent-injector-svc   ClusterIP   10.100.251.30   <none>        443/TCP             40m
service/vault-internal             ClusterIP   None            <none>        8200/TCP,8201/TCP   40m
```
The vault-0 pod is not ready yet because we haven't unsealed Vault. First, expose the vault service so we can connect to it from our laptop. Use the `minikube service` feature:
```
$ minikube service -n vault vault
|-----------|-------|-------------|--------------|
| NAMESPACE | NAME  | TARGET PORT |     URL      |
|-----------|-------|-------------|--------------|
| vault     | vault |             | No node port |
|-----------|-------|-------------|--------------|
😿  service vault/vault has no node port
🏃  Starting tunnel for service vault.
|-----------|-------|-------------|------------------------|
| NAMESPACE | NAME  | TARGET PORT |          URL           |
|-----------|-------|-------------|------------------------|
| vault     | vault |             | http://127.0.0.1:57974 |
|           |       |             | http://127.0.0.1:57975 |
|-----------|-------|-------------|------------------------|
[vault vault  http://127.0.0.1:57974
http://127.0.0.1:57975]
❗  Because you are using a Docker driver on darwin, the terminal needs to be open to run it.
```

As the warning suggests, open a new terminal tab and use the following command to set up the connection. Remember to note the seal key and root token from the output:
```
export VAULT_ADDR='http://127.0.0.1:57974'
vault operator init
```
Use below command or you can paste the URL to browser to unseal it directly (remember to replace your keys):
```
vault operator unseal 1stxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
vault operator unseal 2ndxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
vault operator unseal 3rdxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```
Verify the installation by checking pod:
```
$ kv get pod                                                        
NAME                                   READY   STATUS    RESTARTS   AGE
vault-0                                1/1     Running   0          82m
vault-agent-injector-d986fcb9b-ckrkv   1/1     Running   0          82m
```
Verify by `vault status` command:
```
$ vault status
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    5
Threshold       3
Version         1.16.1
Build Date      2024-04-03T12:35:53Z
Storage Type    file
Cluster Name    vault-cluster-619e93d1
Cluster ID      22768c9b-9983-7beb-9e12-3ac53a825848
HA Enabled      false
```
#### 2. Setup PKI engine as Root CA:
First, log in to Vault using the root token noted earlier. This approach should not be used in a production environment:
```
vault login <root-token>
```
Make sure the PKI engine is enabled in Vault:
```
vault secrets enable -path=pki pki
vault secrets tune -max-lease-ttl=8760h pki
# Generate a root CA certificate
vault write pki/root/generate/internal \
    common_name="tsa-grab-zero-trust.com" \
    ttl=8760h
# Configure the URLs for issuing certificates and CRL distribution points
vault write pki/config/urls \
    issuing_certificates="http://127.0.0.1:57974/v1/pki/ca" \
    crl_distribution_points="http://127.0.0.1:57974/v1/pki/crl"
```
In this scenario, Strimzi will set up an intermediate CA within the `kafka` namespace, and this intermediate CA will be signed by the root CA managed by Vault in the `vault` namespace. We now configure Vault to handle intermediate CA signing requests. So, we need to create a role for issuing intermediate certificates:
```
# Create a role for issuing intermediate certificates
vault write pki/roles/kafka-intermediate \
    allowed_domains="kafka.cluster.local" \
    allow_bare_domains=true \
    allow_subdomains=true \
    max_ttl="43800h"
```
Then, we retrieve the CA certificate from Vault and put it to a secret to prepare for next steps:
```
kubectl create namespace kafka
vault read -field=certificate pki/ca > vault-ca.crt
alias kk='kubectl -n kafka' # make an alias for future queries
kk create secret generic vault-ca-cert --from-file=ca.crt=vault-ca.crt
```