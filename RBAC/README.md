# There are three kinds of users that need access to a Kubernetes cluster
### 1. Developers/Admins
####   Users that are responsible to do administrative or developmental tasks on the cluster. This includes operations like upgrading the cluster or creating the resources/workloads on the cluster

### 2. End Users
####   Users that access the applications deployed on our Kubernetes cluster. Access restrictions for these users are managed by the applications themselves. For example, a web application running on Kubernetes cluster, will have its own security mechanism in place, to prevent unauthorized access

### 3. Applications/Bots
####   There is a possibility that other applications need access to Kubernetes cluster, typically to talk to resources or workloads inside the cluster. Kubernetes facilitates this by using Service Accounts

### create a namespace first
````
$ kubectl create namespace development

namespace/development created
````
### Create client certificate for authentication
#### Since we know that any client can access Kubernetes cluster by authenticating themselves to the kube-apiserver using SSL based authentication mechanism. We will have to generate private key and X-509 client certificate in order to authenticate a user with name DevUser to the kube-apiserver. This user will be working on the development namespace. Let’s create private key and a CSR (Certificate Signing Request) for DevUser

````
$ cd ${HOME}/.kube

$ openssl genrsa -out DevUser.key 2048

Generating RSA private key, 2048 bit long modulus (2 primes)

$ openssl req -new -key DevUser.key -out DevUser.csr -subj "/CN=DevUser/O=development"
````
#### The common name (CN) of the subject will be used as username for authentication request. The organization field (O) will be used to indicate group membership of the user.

#### Once we have private key and CSR, we will have to self sign that CSR to generate the certificate. We will have to provide CA keys of Kubernetes cluster to generate the certificate, as the CA is already approved by minikube/kind cluster.

````
$ openssl x509 -req -in DevUser.csr -CA ${HOME}/.minikube/ca.crt -CAkey ${HOME}/.minikube/ca.key  -CAcreateserial -out DevUser.crt -days 45

Signature ok

subject=CN = DevUser, O = development

Getting CA Private Key
````

#### Please make sure to provide correct set of CA certificate and key. To get to know that, you can run kubectl config view and get the details.

#### Configure kubectl
##### Add entry to users section
````
$ kubectl config set-credentials DevUser --client-certificate ${HOME}/.kube/DevUser.crt --client-key ${HOME}/.kube/DevUser.key

User "DevUser" set.
````

##### Add entry to contexts section
````
$ kubectl config set-context DevUser-context --cluster=minikube --namespace=development --user=DevUser

Context "DevUser-context" created.
````

##### Add more permissions to the user
#### Running kubectl get pods will return the resources of the namespace default for the current context that is minikube. But if we change the context DevUser-context, we will not be able to access the resources. So, running kubectl get pods with new context will result in below error

````
$ kubectl get pods --context=DevUser-context

Error from server (Forbidden): pods is forbidden: User "DevUser" cannot list resource "pods" in API group "" in the namespace "development"
````

#### To enable this newly created user to access to only pods in development namespace, let’s create a role and then bind that role to the DevUser using rolebinding resource. Role is like any other Kubernetes resource. It decides the resources and the actions that someone will be able to take if they have that role. Create a role resource using below manifest

````
$ kubectl apply -f role.yaml
````

#### Bind the role that we have created above to DevUser, using rolebinding resource, with below manifest
````
$ kubectl apply -f rolebingings.yaml
````

#### After creating the role and rolebinding let’s try to list the pods again. We will successfully be able to list them.
````
$ kubectl get pods --context=DevUser-context

No resources found.
we are not able to see any resources because we don't have any pods running in development namespace
````
