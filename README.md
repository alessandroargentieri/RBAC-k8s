# RBAC with Minikube

## Init environment

- go to [Katacoda Minikube environment] (https://www.katacoda.com/courses/kubernetes/launch-single-node-cluster)
- start the minikube cluster with `minikube start`
- wait until the cluster is ready
- create an alias for `kubectl`:

## Check configs

```
kubectl config view
```
You can see how minikube allows to be directly connected to the cluster by using all-permission user `minikube`

**We want to create a new user named `user1` which has no permissions, and then add some reading permissions to him by applying some _yaml_ manifests.

## Create user1

```
# go to home
cd

# generate .key
openssl genrsa -out user1.key 2048

# generate /root/.rnd file for the openssl request (in some envs it doesn't exist)
sudo openssl rand -out /root/.rnd -hex 256

# generate .csr
openssl req -new \
    -key user1.key \
    -out user1.csr \
    -subj "/CN=user1/O=eralabs"

# Check that the files ca.crt and ca.key exist in the location.
ls ~/.minikube/

# generate .crt
openssl x509 -req \
    -in user1.csr \
    -CA ~/.minikube/ca.crt \
    -CAkey ~/.minikube/ca.key \
    -CAcreateserial \
    -out user1.crt \
    -days 500

# Signature ok
# subject=CN = user1, O = eralabs
# Getting CA Private Key    

kubectl config set-credentials user1 \
    --client-certificate=user1.crt \
    --client-key=user1.key

# User "user1" set.

kubectl config set-context user1-context \
    --cluster=minikube \
    --namespace=default \
    --user=user1

# Context "user1-context" created.    

kubectl config view

# response for the config view command
# apiVersion: v1
# clusters:
# - cluster:
#     certificate-authority: /root/.minikube/ca.crt
#     server: https://172.17.0.2:8443
#   name: minikube
# contexts:
# - context:
#     cluster: minikube
#     user: minikube
#   name: minikube
# - context:
#     cluster: minikube
#     namespace: default
#     user: user1
#   name: user1-context
# current-context: minikube
# kind: Config
# preferences: {}
# users:
# - name: minikube
#   user:
#     client-certificate: /root/.minikube/client.crt
#     client-key: /root/.minikube/client.key
# - name: user1
#   user:
#     client-certificate: /root/user1.crt
#     client-key: /root/user1.key


kubectl config use-context user1-context
# Switched to context "user1-context".

kubectl config current-context
# user1-context

kubectl create namespace ns-test # Forbidden
kubectl get pods # Forbidden

# Role & RoleBinding

# Switch back to all-permission user 'minikube'
kubectl config use-context minikube

cat > role.yaml << EOF
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
EOF

kubectl apply -f role.yaml

cat > role-binding.yaml << EOF
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: user1
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f role-binding.yaml

kubectl get roles

kubectl get rolebindings

# switch back to user1 now that he has some more permissions
kubectl config use-context user1-context

kubectl create namespace ns-test # Forbidden

kubectl get pods # Allowed
```

# RBAC with Kubernetes in general 

Go to [Katacoda 2 nodes k8s environment](https://www.katacoda.com/tfogo/scenarios/k8s)

## On Control Plane node

On control plane node we have kubectl installed with admin permissions.

### Generate and sign a certificate for the new user

Let's generate a _key_ and a _certificate_ for user *jean* with the *Certificate Authority* certificate present on the master node:

```bash
openssl genrsa -out jean.key 2048 && \
openssl req -new -key jean.key \
-out jean.csr \
-subj "/CN=jean"

openssl x509 -req -in jean.csr \
-CA /etc/kubernetes/pki/ca.crt \
-CAkey /etc/kubernetes/pki/ca.key \
-CAcreateserial \
-out jean.crt -days 500
```

### Create the new user credentials in k8s

Let's create in k8s the new user through _kubectl_:

```bash
kubectl config set-credentials jean --client-key=jean.key --client-certificate=jean.crt --embed-certs=true

kubectl config set-context jean-context \
--cluster=kubernetes --user=jean
```

### Create and apply roles

Now let's create a new **namespace-reader** role applicable at _namespace_ level by means of the _Role_ k8s API:

```bash
cat > role.yaml << EOF
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: namespace-reader
rules:
- apiGroups: ["*"] # the core API group
  resources: ["deployments", "pods", "services", "persistentvolumes", "persistentvolumeclaims", "statefulsets"]
  verbs: ["get", "watch", "list"]
EOF

kubectl apply -f role.yaml
```

Let's create a new **cluster-reader** role applicable at _cluster_ level by means of the _ClusterRole_ k8s API:

```bash
cat > cluster-role.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-reader
rules:
- apiGroups: ["*"]
  resources: ["persistentvolumes", "namespaces", "secrets"]
  verbs: ["get", "watch", "list"]
EOF

kubectl apply -f cluster-role.yaml
```

### Bind roles to a user

Now that we have created some roles in the cluster, let's bind them to our new user **jean**:

```bash
cat > role-binding.yaml << EOF
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: namespace-reader-binding
  namespace: default
subjects:
- kind: User
  name: jean
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: namespace-reader
  apiGroup: rbac.authorization.k8s.io
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: cluster-reader-binding
subjects:
- kind: User
  name: jean
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-reader
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f role-binding.yaml
```

Now everything on ControlPlane node is ready!

### Open the .kube/configs

Now open the `.kube/configs` file and copy the entire content in order to modify it into a text editor.
Just remove the **admin** users and context to prevent the user to access to all APIs from its local installation of **kubectl**


# On user local machine

The user who wants to access the cluster from his/her local machine must have **kubectl** installed.
Provide him/her the `.kube/config` file got from the ControlPane, after having removed the **admin** user.
It's also important to set in the file:
```yaml
current-context: jean-context
```
in place of the previous admin one.

Enjoy:

```bash
alias k=kubectl
k config view
k config use-context jean-context

k get pods
k get deployments
k create namespace ciao
```
