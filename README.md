# automate-dns
Automate creation of Secrets: 
Kubernetes manifests for the automatic creation of Secrets, implemented on the generated domains of the different components that are part of Elasticsearch, through a Cronjob, using Service Account and RBAC.

## Create Service Account Resource
This resource is useful to authenticate with the Kubernetes API, necessary to execute actions such as CREATE, GET, etc., which will be declared later in the Role resource.
```sh
nano serviceaccount.yaml
kubectl apply -f serviceaccount.yaml
```

## Create the Role Resource
It allows us to declare the permissions that the service account will be able to perform in the assigned namespace, in this case on the secret resource.
```sh
nano  role.yaml
kubectl apply -f role.yaml
```

## Assign Role to Service Account using Role Binding resource
Links the service account to the role, granting permissions in the respective namespace
```sh
nano rb.yaml
kubectl apply -f rb.yaml
```
## Deploy Cronjob
This resource creates a job that we will execute manually, every time that we add a new domain, because of this we will not schedule it. It is made up of two containers, which are executed in order, the first is generated with the ElasticSearch image, fulfilling the function of creating the pk12 files that contain the certificates and private keys, generated from the added domains. The second container generated from the Ubuntu Noble image creates the certificates and private keys through the pk12 files, which it uses to create the Secrets.
```sh
nano final_cronjob.yaml
kubectl apply -f final_cronjob.yaml
```
