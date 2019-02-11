# Tyk + Kubernetes integration

This guide will walk you through a Kubernetes-based Tyk setup. It also covers the required steps to configure and use a Redis cluster and a MongoDB instance. This fork removes use of Google Cloud persistent disks and uses
directory volumes instead for local deployment/testing. Instructions for Tyk Identity Broker have also been added.

# TODO: 
Keycloak setup

# Getting Started

Clone this repository in your workspace:

```
$ cd ~
$ git clone https://github.com/rlankfo/tyk-kubernetes.git
$ cd tyk-kubernetes
```

Make folders for dirctory hostPath volumes:

```
$ cd /mnt
$ mkdir mongo redis-1 redis-2 redis-3 redis-4 redis-5 redis-6 tyk-dashboard tyk-gateway tyk-identity-broker tyk-pump
```

# Redis setup

Enter the `redis` directory:

```
$ cd redis
```

Initialize the `redis` namespace:

```
$ kubectl create -f namespaces
```

Then we import the Redis configuration:

```
$ kubectl create configmap redis-conf --from-file=redis.conf --namespace=redis
```

And initialize the [replica sets](http://kubernetes.io/docs/user-guide/replicasets/) and its pods:

```
$ kubectl create -f replicasets
```

To create the [services](http://kubernetes.io/docs/user-guide/services/) run:

```
$ kubectl create -f services
```

At this point we should have 6 Redis instances, running in each pod. Kubernetes DNS will provide A records for our services, as described [here](http://kubernetes.io/docs/admin/dns/).

## Configuring the cluster

To perform the cluster configuration, we will use a basic Ubuntu pod:

```
$ kubectl run -i --tty ubuntu --image=ubuntu \
    --restart=Never /bin/bash
```

This command may take some time until you get the command prompt.

To install the tools required to configure the cluster, run:

```
$ apt-get update
$ apt-get install -y python2.7 python-pip redis-tools dnsutils
$ pip install redis-trib==0.5.1
```

To configure three masters we use the following command, from the [redis-trib.py](https://github.com/HunanTV/redis-trib.py) tool:

```
$ redis-trib.py create `dig +short redis-1.redis.svc.cluster.local`:6379 \
    `dig +short redis-2.redis.svc.cluster.local`:6379 \
    `dig +short redis-3.redis.svc.cluster.local`:6379
```

Each argument will become the resolved service IP plus the port.

To set a slave for each master we use:

```
$ redis-trib.py replicate --master-addr `dig +short redis-1.redis.svc.cluster.local`:6379 --slave-addr `dig +short redis-4.redis.svc.cluster.local`:6379
$ redis-trib.py replicate --master-addr `dig +short redis-2.redis.svc.cluster.local`:6379 --slave-addr `dig +short redis-5.redis.svc.cluster.local`:6379
$ redis-trib.py replicate --master-addr `dig +short redis-3.redis.svc.cluster.local`:6379 --slave-addr `dig +short redis-6.redis.svc.cluster.local`:6379
```

# MongoDB setup

Enter the `mongo` directory:

```
$ cd ~/tyk-kubernetes/mongo
```

Initialize the Mongo namespace:

```
$ kubectl create -f namespaces
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

## Identity Broker (TIB) setup

Create a config map for `tib.conf`:

```
$ kubectl create configmap tyk-identity-broker-conf --from-file=tib.conf --namespace=tyk
```

Initialize the deployment and service:
```
$ kubectl create -f deployments/tib.yaml
$ kubectl create -f services/tib.yaml
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
