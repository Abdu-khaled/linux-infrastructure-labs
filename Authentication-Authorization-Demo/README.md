# Authentication and Authorization Demo Guide

This exercise guide assumes the following environment, which by default uses the certificate and key from /var/lib/minikube/certs/ , and RBAC mode for authorization:

- Minikube v1.32
- Kubernetes v1.28
- containerd 1.7.8


Start Minikube:

$ minikube start

viem the content of the kubectl client's configuration manifest, observing the only context minikube and the only user minikube, created by default:

$ kubectl config view

```bash
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /home/student/.minikube/ca.crt
    server: ht‌tps://192.168.99.100:8443
  name: minikube
contexts:
- context:
    cluster: minikube
    namespace: default
    user: minikube
  name: minikube
current-context: minikube
kind: Config
preferences: {}
users:
- name: minikube
  user:
    client-certificate: /home/student/.minikube/profiles/minikube/client.crt
    client-key: /home/student/.minikube/profiles/minikube/client.key
```

# Authentication and Authorization Demo Guide (2)


01. Create lfs158 namespace

$ kubectl create namespace lfs158

02. Create the rbac directory and cd into it:
$ mkdir rbac
$ cd rbac

03. Create a new user bob on ur workstaition, and set bob's password as well

~/rbac$ sudo useradd -s /bin/bash bob
~/rbac$ sudo passwd bob

04. Create a private key for the new user bob with the openssl tool, then create a cretificate singing request for bob with the same openssl tool:

~/rbac$ openssl genrsa -out bob.key 2028

~/rbac$ openssl req -new -key bob.key \
   -out bob.csr -subj "/CN=bob=O=learner"

05. Create a YAML definition manifest for a certificate signing request object, and save it with a blank value for the request field: 

```bash 
~/rbac$ vim signing-request.yaml

apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: bob-csr
spec:
  groups:
  - system:authenticated
  request: <assign encoded value from next cat command>
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - digital signature
  - key encipherment
  - client auth
```

# Authentication and Authorization Demo Guide (3)

01. View the certificate, encode it in base64, and assign it to the request filed in the signing-request.yaml file:

~/rbac$ cat bob.csr | base64 | tr -d '/n' , '%'

Copy the output and paste in 'request' field


02. Create the certigicate signing request object, then list the certificate signing request objects. it shows a pending state

~/rbac$ kubectl create -f signing-request.yaml 

~/rbac$ kubectl get csr


03. Approve the certificate signing request object, then list the certificate signing request objects again. It shows both approved and issued states:

~/rbac$ kubectl certificate approve bob-csr

~/rbac$ kubectl get csr

 
# Authentication and Authorization Demo Guide (4)

01. Extract the approved certificate from the certificate signing request, decode it with based64 and save it as certificate file. Then view the certificate in the newly created certificate file:

~/rbac$ kubectl get csr bob-csr \
  -o jsonpath='{.status.certificate}' | \
  base64 -d > bob.crt 

~/rbac$ cat bob.crt


02. Configure the kubectl client's configuration manifest with user bob's credentials by assigning his key and certificate:

~/rbac$ kubectl config set-credentials bob \
   --client-certificate=bob.crt --client-key=bob.key


03. Create a new context entry in the kubectl client's configuration manifest for user bob, associated with the rbac namespace in the minikube cluster:

~/rbac$ kubectl config set-context bob-context \
   --cluster= minikube --namespace=rbac --user=bob



04. View the contents of the kubectl client's configuration manifest again, observing the new context entry bob-context, and the new user entry bob 

~/rbac$ kubectl config view



# Authentication and Authorization Demo Guide (5)

While in the default minikube context, create a new deployment in the rbac namespace:

~/rbac$ kubectl -n lfs158 create deployment nginx --image=nginx:alpine

deployment.apps/nginx created

From the new context bob-context try to list pods. The attempt fails because user bob has no permissions configured for the bob-context:

~/rbac$ kubectl --context=bob-context get pods

Error from server (Forbidden): pods is forbidden: User "bob" cannot list resource "pods" in API group "" in the namespace "lfs158"

The following steps will assign a limited set of permissions to user bob in the bob-context. 


01. Create a YAML configuration manifest for a pod-reader Role object, which allows only get, watch, list actions/verbs in the lfs158 namespace against pod resources. Then create the role object and list it from the default minikube context, but from the rbac namespace:

~/rbac$ vim role.yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: lfs158
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]

~/rbac$ kubectl create -f role.yaml


~/rbac$ kubectl -n lfs158 get roles


# Authentication and Authorization Demo Guide (6)

01. Create a YAML configuration manifest for a rolebinding object, which assigns the permissions of the pod-reader Role to user bob. Then create the rolebinding object and list it from the default minikube context, but from the rbac namespace:

~/rbac$ vim rolebinding.yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-read-access
  namespace: lfs158
subjects:
- kind: User
  name: bob
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

~/rbac$ kubectl create -f rolebinding.yaml 

~/rbac$ kubectl -n rbac get rolebindings

Now that we have assigned permissions to bob, we can successfully list pods from the new context bob-context.

~/rbac$ kubectl --context=bob-context get pods

'Important: make sure the crt issued for the right cluster you need'










