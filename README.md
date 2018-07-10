# Tyk + Kubernetes integration

This guide will walk you through a Kubernetes-based Tyk setup on your local machine. 
It also provides direction on deployment of a simple redis-server & mongo-db instance.

# Requirements

We shall assume that you already have kubectl &ampl minikube installed / configured.

# Getting Started

Clone this repository in your workspace:

```
git clone git@github.com:TykTechnologies/tyk-kubernetes.git && cd tyk-kubernetes
```

## Prerequisites

Tyk-Pro requires both Redis & MongoDB - we shall run through a quick installation to get you started, however please
note that this is a quickstart - and as such, not suitable for production use.

### Redis installation

For the purposes of this installation guide, we shall deploy a single-node redis-server.

```
kubectl apply -f ./redis/redis.yaml
```

## MongoDB setup

Enter the `mongo` directory:

```
$ cd ~/tyk-kubernetes/mongo
```

Initialize the Mongo namespace:

```
$ kubectl create -f namespaces
```

Create a volume for MongoDB:

```
$ gcloud compute disks create --size=10GB mongo-volume
```

Initialize the deployment and service:

```
$ kubectl create -f deployments
$ kubectl create -f services
```

# Tyk setup

Enter the `tyk` directory:

```
$ cd ~/tyk-kubernetes/tyk
```

Initialize the Tyk namespace:

```
$ kubectl create -f namespaces
```

## Dashboard setup

Create a volume for the dashboard:

```
$ gcloud compute disks create --size=10GB tyk-dashboard
```

Set your license key in `tyk_analytics.conf`:

```json
    "mongo_url": "mongodb://mongodb.mongo.svc.cluster.local:27017/tyk_analytics",
    "license_key": "LICENSEKEY",
```

Then create a config map for this file:

```
$ kubectl create configmap tyk-dashboard-conf --from-file=tyk_analytics.conf --namespace=tyk
```

Initialize the deployment and service:

```
$ kubectl create -f deployments/tyk-dashboard.yaml
$ kubectl create -f services/tyk-dashboard.yaml
```

Check if the dashboard has been exposed:

```
$ kubectl get service tyk-dashboard --namespace=tyk
```

The output will look like this:

```
NAME            CLUSTER-IP       EXTERNAL-IP       PORT(S)          AGE
tyk-dashboard   10.127.248.206   x.x.x.x           3000:30930/TCP   45s
```

`EXTERNAL-IP` represents a public IP address allocated for the dashboard service, you may try accessing the dashboard using this IP, the final URL will look like: 

```
http://x.x.x.x:3000/
```

At this point we should bootstrap the dashboard, we need to locate the pod that's running the dashboard:

```
$ kubectl get pod --namespace=tyk
NAME                             READY     STATUS    RESTARTS   AGE
tyk-dashboard-1616536863-oeqa2   1/1       Running   0          2m
```

Then we run the bootstrap script, using the pod name:

```
$ kubectl exec --namespace=tyk tyk-dashboard-1616536863-oeqa2 /opt/tyk-dashboard/install/bootstrap.sh x.x.x.x
```

Remember to use the `EXTERNAL-IP` that shows up in the previous step, instead of `x.x.x.x`.

The bootstrap script will report the initial credentials:

```
DONE
====
Login at http://x.x.x.x:3000/
User: test@test.com
Pass: test123
```

You should be able to access the dashboard now.

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