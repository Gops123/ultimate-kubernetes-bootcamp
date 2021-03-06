# Kubernetes Access Control:  Authentication and Authorization

In  this lab you are going to,

  * Create users and groups and setup certs based authentication
  * Create service accounts for applications
  * Create Roles and ClusterRoles to define authorizations
  * Map Roles and ClusterRoles to subjects i.e. users, groups and service accounts using RoleBingings and ClusterRoleBindings.


## How one can access the Kubernetes API?

The Kubernetes API can be accessed by three ways.

  * Kubectl - A command line utility of Kubernetes
  * Client libraries - Go, Python, etc.,
  * REST requests

## Who can access the Kubernetes API?

Kubernetes API can be accessed by,

  * Human Users
  * Service Accounts  

Each of these topics will be discussed in detail in the later part of this chapter.

## Stages of a Request

When a request tries to contact the API , it goes through various stages as illustrated in the image given below.

  ![request-stages](images/access-control.svg)
  <sub>[source: official kubernetes site](https://kubernetes.io/docs/home/)</sub>


### Stage 1: Authentication

  * Authentication operation checks whether the *user/service account* has the permission to talk to the api server or not.  
  * Authentication is done by the authentication modules which are configured with the api server.  
  * Cluster uses with one or more authentication modules enabled.  
  * If the request fails to authenticate itself, it will be served with **401 error**.  

#### Authentication for Human  Users

  * Kubernetes uses **usernames** for access control.  
  * But it neither has an api object nor stores information about users in its data store.  
  * Users need to be managed externally by the cluster administrator.  

#### Authentication for Service Accounts

  * Unlike user accounts, service accounts are managed by Kubernetes.  
  * *service accounts* are bound to specific namespaces.  
  * Credentials for *service Accounts* are stored as *secrets*.  
  * These secrets are mounted to pods when a deployment starts using the Service Account.  

### Stage 2: Authorization

  * After a request successfully authenticated, it goes through the authorization process.
  * In order for a request to be authorized, it must consist following attributes.
    * Username of the requester(User)
    * Requested action(Verb)
    * The object affected(Resource)
  * Authorization is done by the following modules. Each of these modules has a special purpose.
    * Attribute Based Access Control(ABAC)
    * Role Based Access Control(RBAC)
    * Node Authorizer
    * Webhook module
  * If a request is failed to get authorized, it will be served with **403 error**.
  * Among these modules, RBAC is the most used authorizer while,
    * ABAC is used for,
      * Policy based, fine grained access control
      * The caveat is api server has to be restarted whenever we define a ABAC policy
    * Node Authorizer is,
      * Enabled in all the worker nodes
      * Grants access to kubelet for some of the resources.
  * We have already talked about the user in detail. Now lets focus on **verbs** and **resources**
  * We will talk about RBAC in detail in the later part

### Stage 3: Admission Control
  * Admission control part is taken care of by the software modules that can modify/reject requests.
  * Admission control is mainly used for fine-tuning access control.
  * Admission control can directly act on the object being modified.


## Role Based Access Control (RBAC)

| Group     | User     |  Namespace     |  Resources     |  
| :------------- | :------------- |  :------------- | :------------- |   
| ops       | maya       | all | all |  
| dev       | kim       | instavote | pods, deployments, statefulsets, configmaps, secrets, services, jobs, crons  |  


Generate the user's private key
```
openssl genrsa -out john.key 2048
openssl req -new -key john.key -out john.csr -subj "/CN=john/O=initcron"
```
Sign the key using kubernetes CA
`Before proceeding further, please note that, you will need the server's ca certificate and ca key in your local machine.`
```
openssl x509 -req -in john.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out john.crt -days 500
```
Save these newly generated keys in a secure directory of the user's system (Ex: /home/john/.kube-certs)
Execute the following commands on the users system
```
kubectl config set-credentials john --client-certificate=/home/john/.kube-certs/john.crt --client-key=/home/john/.kube-certs/john.key
kubectl config set-context john-context --cluster=<YOUR-CLUSTER-NAME> --namespace=<NAMESPACE-FOR-USER>  --user=john
# This step will throw an error
kubectl --context=john-context get pods
```

## Create role for the developers
These steps (3,4) has to be executed from kube-master or the controller machine
`dev-role.yml`
```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: dev
  name: developer
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["deployments", "replicasets", "pods"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```
```
kubectl create -f dev-role.yml
```

## Create RoleBindings for the dev-role
`dev-rolebinding.yml`
```
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: developer
  namespace: developer
subjects:
- kind: User
  name: john
  apiGroup: ""
roleRef:
  kind: Role
  name: developer
  apiGroup: ""
```
```
kubectl create -f dev-rolebinding.yaml
```


### TLS in Kubernetes
Kubernetes API typically runs on two ports.
  * 8080
    * This port is available only for processes running on master machine (kube-scheduler, kube-controller-manager).
    * This port is insecure.
  * 6443
    * TLS Enabled.
    * Available for kubectl and others.


#### Verbs
  * Verbs are the **action** to be taken on resources.
  * Some of the verbs in Kubernetes are,
    * get
    * list
    * create
    * update
    * patch
    * watch
    * delete

#### Resources
  * Resources are the object being manipulated by the verb.
  * Ex: pods, deployments, service, namespaces, nodes, etc.,


## Role Based Access Control (RBAC)
  * RBAC is the most used form of access control to **grant or revoke** permissions to users.
  * It is used for dynamically configuring policies through API.
  * RBAC has two objects that are used for creating polices and another two objects that are used for implementing those policies.
  * Objects that are used to create policies,
    * Roles,
    * ClusterRoles.
  * Objects that are used to implement policies,
    * RoleBindings,
    * ClusterRoleBindings.

### Roles
  * Roles grant access to resources **within a single namespace**.
  * Roles cannot grant permission for global(cluster-wide) resources.

### ClusterRoles
  * ClusterRoles works similar to Roles, but for **cluster-wide** resources.

### RoleBindings
  * RoleBidings are used to grant permission defined in a Role to **a user or a set of users**.
  * RoleBindings can also refer to *ClusterRoles*.

### ClusterRoleBindings
  * ClusterRoleBindings works same as RoleBindings, but cluster-wide.
