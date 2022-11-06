---
title: "Teslamate in Kubernetes"
date: 2022-10-18T17:40:46Z
draft: false
---

## Overview

This post outlines my configuration for running the wonderful [Teslamate](https://github.com/adriankumpf/teslamate) application stack within Kubernetes.

## Assumptions

* I use NFS-backed storage in Kubernetes k3s (located on TrueNAS at `/mnt/nvme/k8s_storage` and shared via NFS at `nas01.lan:/mnt/nvme/k8s_storage`). You'll need to modify your volume details if using another storage option.
* I use `metallb` on-prem. Your LoadBalancer solution (whether on-premises or in the cloud) should conform to this setup, as the Kube API is very standardized on that front. But just an FYI, you may need to reference your LoadBalancer solution docs.
* I have an existing MQTT setup external to this setup. If you need to add MQTT to this, you can reference my guide on MQTT: [Blog: Mosquitto MQTT in Kubernetes](/mosquitto-mqtt-in-kubernetes)

## Process

First off, we will create our folder structure locally on our controller node and in our storage solution:

``` shell
root@k8s04:~/homelab/k3s# mkdir teslamate
```

``` shell
user@truenas:~$ mkdir /mnt/k8s_storage/teslamate
user@truenas:~$ mkdir /mnt/k8s_storage/teslamate/import
user@truenas:~$ mkdir /mnt/k8s_storage/teslamate/db
user@truenas:~$ mkdir /mnt/k8s_storage/teslamate/grafana
```

Now we get to the Kube details:

`namespaces.yml`:

``` yml
apiVersion: v1
kind: Namespace
metadata:
  name: teslamate
```

`services.yml`:

``` yml
---
apiVersion: v1
kind: Service
metadata:
  name: teslamate-app
  namespace: teslamate
spec:
  selector:
    app.kubernetes.io/name: teslamate-app
  ports:
  - name: teslamate-app
    protocol: TCP
    port: 4000
    targetPort: http
  type: LoadBalancer
  loadBalancerIP: 10.1.10.153
  externalTrafficPolicy: Local

---
apiVersion: v1
kind: Service
metadata:
  name: teslamate-db
  namespace: teslamate
spec:
  selector:
    app.kubernetes.io/name: teslamate-db
  ports:
  - name: teslamate-db
    protocol: TCP
    port: 5432
    targetPort: db
  type: LoadBalancer
  loadBalancerIP: 10.1.10.154
  externalTrafficPolicy: Local

---
apiVersion: v1
kind: Service
metadata:
  name: teslamate-grafana
  namespace: teslamate
spec:
  selector:
    app.kubernetes.io/name: teslamate-grafana
  ports:
  - name: teslamate-grafana
    protocol: TCP
    port: 3000
    targetPort: grafana
  type: LoadBalancer
  loadBalancerIP: 10.1.10.155
  externalTrafficPolicy: Local
```

`deployments.yml`:

``` yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: teslamate
  namespace: teslamate
  labels:
    app.kubernetes.io/name: teslamate
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: teslamate
  template:
    metadata:
      labels:
        app.kubernetes.io/name: teslamate
    spec:
      containers:
      - name: teslamate
        image: teslamate/teslamate:latest
        ports:
          - containerPort: 4000
            name: http
        volumeMounts:
          - name: teslamate-import
            mountPath: /opt/app/import
        env:
        - name: DATABASE_USER
          value: 'teslamate'
        - name: DATABASE_PASS
          value: 'secret'
        - name: DATABASE_NAME
          value: 'teslamate'
        - name: DATABASE_HOST
          value: 'teslamate-db.teslamate'
        - name: MQTT_HOST
          value: 'mqtt.lan' # <<-- Change this to your MQTT host
        - name: ENCRYPTION_KEY
          value: somekeytext
      volumes:
        - name: teslamate-import
          nfs: 
            server: nas01.lan
            path: /mnt/nvme/k8s_storage/teslamate/import

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: teslamate-db
  namespace: teslamate
  labels:
    app.kubernetes.io/name: teslamate-db
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: teslamate-db
  template:
    metadata:
      labels:
        app.kubernetes.io/name: teslamate-db
    spec:
      containers:
      - name: teslamate-db
        image: postgres:14
        ports:
          - containerPort: 5432
            name: db
        volumeMounts:
          - name: teslamate-db
            mountPath: /var/lib/postgresql/data
        env:
        - name: POSTGRES_USER
          value: 'teslamate'
        - name: POSTGRES_PASSWORD
          value: 'secret'
        - name: POSTGRES_DB
          value: 'teslamate'
      volumes:
        - name: teslamate-db
          nfs: 
            server: nas01.lan
            path: /mnt/nvme/k8s_storage/teslamate/db

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: teslamate-grafana
  namespace: teslamate
  labels:
    app.kubernetes.io/name: teslamate-grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: teslamate-grafana
  template:
    metadata:
      labels:
        app.kubernetes.io/name: teslamate-grafana
    spec:
      containers:
      - name: teslamate-grafana
        image: teslamate/grafana:latest
        ports:
          - containerPort: 3000
            name: grafana
        volumeMounts:
          - name: teslamate-grafana
            mountPath: /var/lib/grafana
        env:
        - name: DATABASE_USER
          value: 'teslamate'
        - name: DATABASE_PASS
          value: 'secret'
        - name: DATABASE_NAME
          value: 'teslamate'
        - name: DATABASE_HOST
          value: 'teslamate-db.teslamate'
      volumes:
        - name: teslamate-grafana
          nfs: 
            server: nas01.lan
            path: /mnt/nvme/k8s_storage/teslamate/grafana
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
