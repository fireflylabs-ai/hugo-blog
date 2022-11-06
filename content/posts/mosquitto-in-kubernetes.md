---
title: "Mosquitto MQTT in Kubernetes"
date: 2022-10-18T17:44:04Z
draft: false
---

## Overview

This post outlines my configuration for running an MQTT stack within Kubernetes for all your IOT messaging needs.

## Assumptions

* I use NFS-backed storage in Kubernetes k3s (located on TrueNAS at `/mnt/nvme/k8s_storage` and shared via NFS at `nas01.lan:/mnt/nvme/k8s_storage`). You'll need to modify your volume details if using another storage option.
* I use `metallb` on-prem. Your LoadBalancer solution (whether on-premises or in the cloud) should conform to this setup, as the Kube API is very standardized on that front. But just an FYI, you may need to reference your LoadBalancer solution docs.
* My MQTT configuration is very barebones and is good for my use-case. Please consult the [Mosquitto.conf docs](https://mosquitto.org/man/mosquitto-conf-5.html) for other configuration options.

## Process

First off, we will create our folder structure locally on our controller node and in our storage solution:

``` shell
root@k8s04:~/homelab/k3s# mkdir mqtt
```

``` shell
user@truenas:~$ mkdir /mnt/k8s_storage/mqtt
user@truenas:~$ mkdir /mnt/k8s_storage/mqtt/config
user@truenas:~$ mkdir /mnt/k8s_storage/mqtt/data
user@truenas:~$ mkdir /mnt/k8s_storage/mqtt/log
```

Next, we will populate our mosquitto.conf file at `/mnt/k8s_storage/mqtt/config` with some baseline config:

``` text
listener 1883 0.0.0.0   # Sets MQTT to listen on all IPs and port 1883
persistence true        # Tells MQTT to store messages to disk
persistence_file mosquitto.db           # Filename to store messages on disk
persistence_location /mosquitto/data    # Folder location to store messages on disk.
log_dest stdout         ##########
log_dest topic          #
log_type error          #
log_type warning        #  Optional logging
log_type notice         #
log_type information    #
log_timestamp true      #########
allow_anonymous true    # Allow anonymous access (no username/password required)
```

Now we get to the Kube details:

`mqtt/namespaces.yml`:

``` yml
apiVersion: v1
kind: Namespace
metadata:
  name: mqtt
```

`mqtt/deployments.yml`:

``` yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mqtt
  namespace: mqtt
  labels:
    app.kubernetes.io/name: mqtt
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: mqtt
  template:
    metadata:
      labels:
        app.kubernetes.io/name: mqtt
    spec:
      containers:
      - name: mqtt
        image: eclipse-mosquitto
        ports:
          - containerPort: 1883
            name: mqtt-app
        volumeMounts:
          - name: mqtt-data
            mountPath: /mosquitto
      volumes:
        - name: mqtt-data
          nfs: 
            server: nas01.lan
            path: /mnt/nvme/k8s_storage/mqtt
```

`mqtt/services.yml`:

``` yml
apiVersion: v1
kind: Service
metadata:
  name: mqtt
  namespace: mqtt
spec:
  selector:
    app.kubernetes.io/name: mqtt
  ports:
  - name: mqtt-app
    protocol: TCP
    port: 1883
    targetPort: mqtt-app
  type: LoadBalancer
  loadBalancerIP: 10.1.10.152 # Change this as needed for your setup
  externalTrafficPolicy: Local
```

Wrap it all up with a bow via `kustomize.yml`:

``` yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - namespaces.yml
  - services.yml
  - deployments.yml
```

You can fire off your deployment via:

``` shell
root@k8s04:~/homelab/k3s# kubectl apply -k mqtt/
namespace/mqtt created
service/mqtt created
deployment.apps/mqtt created
```

And confirm it's all functioning via:

``` shell
root@k8s04:~/homelab/k3s# kubectl get all -n mqtt
NAME                        READY   STATUS    RESTARTS   AGE
pod/mqtt-5f46c7558c-6cfn7   1/1     Running   0          58s

NAME           TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/mqtt   LoadBalancer   10.43.183.191   10.1.10.152   1883:30778/TCP   59s

NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mqtt   1/1     1            1           59s

NAME                              DESIRED   CURRENT   READY   AGE
replicaset.apps/mqtt-5f46c7558c   1         1         1       59s
```

MQTT is now available at `10.1.10.152:1883`.
