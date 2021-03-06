# Larakube

Local *Minikube* development with Laravel can be done with this Larakube setup. This setup has the Minikube Ingress Nginx, PHP FPM, Nginx, MySQL, Redis as well as storage deployments using custom as well as default images.

**NB** Does not play well with Laravel Valet due to its use of dnsmasq. Turn off DNS Masq with `sudo brew services stop dnsmasq` before using Minikube.

## Requirements

- Docker
- Minikube
- VM installed as part of Minikube such as Hyperkit
- Dnsmasq turned off or customized
- `/etc/hosts` tweak to load site locally (`minikube ip` for ip to add to hosts file)

## Quick Start

To quickly get started:

- rename `secrets//mysql-secrets.example` to `secrets/.mysql-secrets.yml`
- add values to `secrets/mysql-secrets.yml`
- add values to `secrets/mysql-secrets.yml`
- rename `configs/laravel_configMap.yml.example` to `configs/laravel_configMap.yml`
- add your own values
- add values to `configs/mysql_configMap.yml` of your own choosing
- run `minikube start` 

followed by the use of the `kube.sh` script in the local directory:

- on Zsh shell use `bash kube.sh`
- on Bash shell use `./kube.sh`


It will run all the necessary scripts to get you started with Minikube locally. If you prefer to follow the process some more and do things more hands on then continue reading the steps below.

**NB** Due to Ingress setup being somewhat slow sometimes you may need to run script twice.

## Startup

As with the quickstart do do the following:

- rename `secrets//mysql-secrets.example` to `secrets/.mysql-secrets.yml`
- add values to `secrets/mysql-secrets.yml`
- add values to `secrets/mysql-secrets.yml`
- rename `configs/laravel_configMap.yml.example` to `configs/laravel_configMap.yml`
- add your own values
- add values to `configs/mysql_configMap.yml` of your own choosing


To get Minikube running execute the following command:

```
minikube start
```

**NB** we mount our data volume right away so we can use it later on for our storage.

Then to check and make sure you have the proper context up and running do a

```
kubectl config current-context
```

It should show *minikube*

### Local Namespace

To create a namespace based on file you can use this:

```
kubectl apply -f namespace.yml
```

Then this namespace can be used instead of default to launch your pods into. To check whethere the new namespace is there you can run

```
kubectl get namespaces
```
to make namespace default use 

```
kubectl config set-context --current --namespace=smt-local
```

### Local Ingress Controller

Locally you can run an Ingress Nginx as well, but in a slightly different way

https://kubernetes.io/docs/tasks/access-application-cluster/ingress-minikube/

##### Enable Minikube Nginx Ingres 
To enable the NGINX Ingress controller, run the following command:

```
minikube addons enable ingress
```

Verify that the NGINX Ingress controller is running:

```
kubectl get pods -n kube-system
```

### Ingress Resource 

https://kubernetes.io/docs/tasks/access-application-cluster/ingress-minikube/#create-an-ingress-resource

To create a resource for your Ingress Nginx Controller to send traffic to your Service we need to set this up.

```
kubectl apply -f services/ingress.yml
```

Once this is up and running you can check for address and port with 

```
kubectl get ingress
Warning: extensions/v1beta1 Ingress is deprecated in v1.14+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
NAME               CLASS    HOSTS            ADDRESS        PORTS   AGE
ingress-resource   <none>   smart48k8.test   192.168.64.5   80      6m48s
```

**NBB** We added a host in this file called `smart48k8.local` and you do need to check `minikube ip` to attach this host to the ip in `/etc/host` for it to load.

### Local Persistent Volume

resources:
- https://minikube.sigs.k8s.io/docs/handbook/persistent_volumes/
- https://stackoverflow.com/questions/45511339/kubernetes-minikube-with-local-persistent-storage

to use the storage for local testing apply the one in local directory 

_minikube supports PersistentVolumes of type hostPath out of the box. These PersistentVolumes are mapped to a directory inside the running minikube instance (usually a VM, unless you use --driver=none, --driver=docker, or --driver=podman). For more information on how this works, read the Dynamic Provisioning section below._

_In addition, minikube implements a very simple, canonical implementation of dynamic storage controller that runs alongside its deployment. This manages provisioning of hostPath volumes (rather then via the previous, in-tree hostPath provider)._
#### Persisent volume claim


We will use the default dynamic storage class so no need to create volumes:

_...canonical implementation of dynamic storage controller that runs alongside its deployment._

https://platform9.com/blog/tutorial-dynamic-provisioning-of-persistent-storage-in-kubernetes-with-minikube/


```
kubectl apply -f storage/code-pv-claim.yml
kubectl apply -f storage/nginx-pv-claim.yml
kubectl apply -f storage/mysql-pv-claim.yml
kubectl apply -f storage/redis-pv-claim.yml
```

### Persistent Volumes' Setup

and to check it has been created and is running we can use 

```
kubectl get pv
``` 

and to delete all (dangerous) use `kubectl delete pvc --all`

*Notes on Persistent volume and testing still needed*

To check where all is stored do a pcv check (tab in zsh allows choosing on of them):

```
kubectl describe pv pvc-.....
```

to see the Path. In the case of Redis with will be `/tmp/hostpath-provisioner/smt-local/redis-pv-claim` Then ssh into minikube using 

```
minkube ssh
```

and see:


```
cd /tmp/hostpath-provisioner/smt-local/              
$ ls -la
total 20
drwxr-xr-x 5 root root 4096 Dec  7 05:02 .
drwxr-xr-x 3 root root 4096 Dec  7 05:01 ..
drwxrwxrwx 2 root root 4096 Dec  7 06:57 code-pv-claim
drwxrwxrwx 7  999 root 4096 Dec  8 06:39 mysql-pv-claim
drwxrwxrwx 2 root root 4096 Dec  7 05:02 nginx-pv-claim
```

### Database Secrets & Config Vars

To implmenet the known parts that do not need to be secret apply the MySQL Config map:

```
kubectl apply -f configs/mysql_configMap.yml
kubectl apply -f configs/redis_configMap.yml
```

We do have secrets to store MySQL data. Do rename the file / remove the `.example` part before you run this command:

```
kubectl apply -f secrets/mysql-secrets.yml
```

### Services 

To have Ingress load data from the web deployment where we have Nginx and PHP FPM you need one service:

```
kubectl apply -f services/web.yml
```

For the workspace we use the following:

```
kubectl apply -f services/workspace.yml
```

We may remove this service later down the line.

And then we have the MySQL and Redis services to connect to the databases. We will implement two MySQL database services in the future but we start with one:

```
kubectl apply -f services/mysql.yml
kubectl apply -f services/redis.yml

```
### Deployments 

Local deployments are split in deployments for the app and other containers

To fire up the app with the Laravel and Nginx container 

- rename `configs/laravel_configMap.yml.example` to `configs/laravel_configMap.yml` if you hadn't
- add your own values

```
kubectl apply -f configs/laravel_configMap.yml
```

Then run the Nginx configuration file


```
kubectl apply -f configs/nginx_configMap.yaml
```

followed by the actual deployment
```
kubectl apply -f deployments/web.yml
```
#### Workspace and Worker Deployments

then we have the other deployments excluding the databases:

```
kubectl apply -f deployments/php-worker.yml
kubectl apply -f deployments/workspace.yml
```

#### Workspace

Workspace currently works well with NPM and composer. First setup has all files and directories as root however. So a `chown -R laradock:laradock code/` is necessry to make them owned by user and group laradock which translates into user docker and group docker on the VM. This is still not ideal but better.
### Database Deployments

To run the MySQL database and Redis containers run

```
kubectl apply -f deployments/mysql.yml
kubectl apply -f deployments/redis.yml
```
