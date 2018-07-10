# Tyk + Kubernetes integration

This guide will walk you through a Kubernetes-based Tyk setup on your local machine. 
It also provides direction on deployment of a simple redis-server & mongo-db instance.

# Requirements

We shall assume that you already have kubectl &ampl minikube installed / configured.

Please note that this is simply a guide to getting Tyk installed quickly on Kubernetes. It is not production ready.

# Getting Started

Clone this repository in your workspace:

```
git clone git@github.com:TykTechnologies/tyk-kubernetes.git && cd tyk-kubernetes
```

## Databases

Tyk-Pro requires both Redis & MongoDB - we shall run through a quick installation to get you started, with single
 instances of each.

### Redis installation

For the purposes of this installation guide, we shall deploy a single-node redis-server.

```
kubectl apply -f ./redis/redis.yaml
```

Check that the Redis is up and running:

```
kubectl get pods -l app=redis
NAME                     READY     STATUS    RESTARTS   AGE
redis-5f8bc7f679-qk7cw   1/1       Running   0          38m
```

Check the logs for the redis pod

```
kubectl logs redis-5f8bc7f679-qk7cw
1:C 10 Jul 15:32:07.875 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:C 10 Jul 15:32:07.875 # Redis version=4.0.10, bits=64, commit=00000000, modified=0, pid=1, just started
1:C 10 Jul 15:32:07.875 # Configuration loaded
1:M 10 Jul 15:32:07.877 * Running mode=standalone, port=6379.
1:M 10 Jul 15:32:07.877 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
1:M 10 Jul 15:32:07.877 # Server initialized
1:M 10 Jul 15:32:07.877 * Ready to accept connections
```

### MongoDB installation

For the purposes of this installation, we will deploy a single mongodb server.

```
kubectl apply -f ./mongo/mongo.yaml
```

Check that mongo pod is up and running

```
kubectl get pods -l app=mongodb
NAME                       READY     STATUS    RESTARTS   AGE
mongodb-68d66b5d69-r6cxq   1/1       Running   0          14m
```

Check logs for mongodb

```
kubectl logs mongodb-68d66b5d69-r6cxq
2018-07-10T16:01:46.464+0000 I CONTROL  [initandlisten] MongoDB starting : pid=1 port=27017 dbpath=/data/db 64-bit host=mongodb-68d66b5d69-r6cxq
2018-07-10T16:01:46.465+0000 I CONTROL  [initandlisten] db version v3.4.15
2018-07-10T16:01:46.465+0000 I CONTROL  [initandlisten] git version: 52e5b5fbaa3a2a5b1a217f5e647b5061817475f9
2018-07-10T16:01:46.465+0000 I CONTROL  [initandlisten] OpenSSL version: OpenSSL 1.0.1t  3 May 2016
2018-07-10T16:01:46.465+0000 I CONTROL  [initandlisten] allocator: tcmalloc
2018-07-10T16:01:46.465+0000 I CONTROL  [initandlisten] modules: none
2018-07-10T16:01:46.465+0000 I CONTROL  [initandlisten] build environment:
2018-07-10T16:01:46.465+0000 I CONTROL  [initandlisten]     distmod: debian81
2018-07-10T16:01:46.465+0000 I CONTROL  [initandlisten]     distarch: x86_64
2018-07-10T16:01:46.465+0000 I CONTROL  [initandlisten]     target_arch: x86_64
2018-07-10T16:01:46.465+0000 I CONTROL  [initandlisten] options: { storage: { mmapv1: { smallFiles: true } } }
...
```

## Tyk Dasboard

First, we need to get your license key & configs into K8S. For brevity, we will just store this secret inside a 
configmap, however they should probably be stored as secrets.

Replace `DASHBOARD_LICENSE` in `dashboard/dashboard.yaml` with your license key, then initialise and deploy the dashboard.

```
kubectl apply -f dashboard/dashboard.yaml
```

Check the logs of one of the dashboard pods to ensure all is running smoothly:

```
$ kubectl get pods -l app=tyk-dashboard
NAME                             READY     STATUS    RESTARTS   AGE
tyk-dashboard-5f6ff6859c-prlzx   1/1       Running   0          12m
tyk-dashboard-5f6ff6859c-xwf7b   1/1       Running   0          12m

$ kubectl logs tyk-dashboard-5f6ff6859c-prlzx
time="Jul 10 20:24:32" level=info msg="Using /etc/tyk-dashboard/tyk_analytics.conf for configuration" 
time="Jul 10 20:24:32" level=info msg="connecting to MongoDB: [mongodb.default.svc.cluster.local:27017]" 
time="Jul 10 20:24:32" level=info msg="mongo connection established" 
time="Jul 10 20:24:32" level=info msg="Creating new Redis connection pool" 
time="Jul 10 20:24:32" level=info msg="Creating new Redis connection pool" 
time="Jul 10 20:24:32" level=info msg="Creating new Redis connection pool" 
time="Jul 10 20:24:32" level=info msg="Creating new Redis connection pool" 
time="Jul 10 20:24:32" level=info msg="Adding available nodes..." 
time="Jul 10 20:24:32" level=info msg="Tyk Analytics Dashboard v1.6.2" 
time="Jul 10 20:24:32" level=info msg="Copyright Martin Buhr 2016" 
time="Jul 10 20:24:32" level=info msg="https://www.tyk.io" 
time="Jul 10 20:24:32" level=info msg="Listening on port: 3000" 
time="Jul 10 20:24:32" level=info msg="Registering nodes..." 
time="Jul 10 20:24:32" level=info msg="Adding available nodes..." 
time="Jul 10 20:24:32" level=info msg="Creating new Redis connection pool" 
time="Jul 10 20:24:32" level=info msg="Socket server started" 
time="Jul 10 20:24:32" level=info msg="--> Standard listener (http) for UI notifications" addr=":5000" 
time="Jul 10 20:24:32" level=info msg="--> Standard listener (http) for dashboard and API" 
time="Jul 10 20:24:32" level=info msg="Starting zeroconf heartbeat" 
time="Jul 10 20:24:32" level=info msg="Starting notification handler for gateway cluster" 
time="Jul 10 20:24:32" level=info msg="Loading routes..." 
```

We can bootstrap the dashboard by targeting one of them by it's pod name:

```
kubectl exec tyk-dashboard-5f6ff6859c-prlzx /opt/tyk-dashboard/install/bootstrap.sh 127.0.0.1
```

// TODO Find way to bootstrap dash with `$(minikube ip)` rather than `127.0.0.1`.

The bootstrap script will report the initial credentials:

```
DONE
====
Login at http://127.0.0.1:3000/
User: test@test.com
Pass: test123
``` 

You should be able to access the dashboard now. But because you are running inside Kubernetes, you will not be able to
access via the advertised login `http://127.0.0.1:3000/`. Instead, the dashboard will be available via address provided
by minikube:

```
$ minikube service list
|-------------|----------------------|-----------------------------|
|  NAMESPACE  |         NAME         |             URL             |
|-------------|----------------------|-----------------------------|
| default     | kubernetes           | No node port                |
| default     | mongodb              | No node port                |
| default     | redis                | No node port                |
| default     | tyk-dashboard        | http://192.168.99.100:30001 |
| kube-system | kube-dns             | No node port                |
| kube-system | kubernetes-dashboard | http://192.168.99.100:30000 |
|-------------|----------------------|-----------------------------|
```

## Gateway setup

Create a volume for the gateway:

```
$ gcloud compute disks create --size=10GB tyk-gateway
```

Create a config map for `tyk.conf`:

```
$ kubectl create configmap tyk-gateway-conf --from-file=tyk.conf --namespace=tyk
```

Initialize the deployment and service:

```
$ kubectl create -f deployments/tyk-gateway.yaml
$ kubectl create -f services/tyk-gateway.yaml
```

## Pump setup

Create a config map for `pump.conf`:

```
$ kubectl create configmap tyk-pump-conf --from-file=pump.conf --namespace=tyk
```
Initialize the pump deployment:

```
$ kubectl create -f deployments/tyk-pump.yaml
```

# FAQ

## Which services does this setup expose?

The `tyk-gateway` and `tyk-dashboard` services are publicly exposed, using load balanced IPs provisioned by Google Cloud Platform. See the `type` key in `tyk/services/tyk-dashboard.yaml` and `tyk/services/tyk-gateway.yaml`. For more advanced setups you may check [this guide](https://cloud.google.com/container-engine/docs/tutorials/http-balancer).

## How do I check the Tyk logs?

To check the logs you must use a specific pod name, first list the pods available under the `tyk` namespace:

```
$ kubectl get pod --namespace=tyk
NAME                             READY     STATUS    RESTARTS   AGE
tyk-dashboard-1616536863-zmpwb   1/1       Running   0          8m
...
```

Then request the logs:

```
$ kubectl logs tyk-dashboard-1616536863-zmpwb
```

## How do I update the Tyk configuration?

You must replace the Tyk [config maps](http://kubernetes.io/docs/user-guide/configmap/) and recreate or restart the Tyk services.