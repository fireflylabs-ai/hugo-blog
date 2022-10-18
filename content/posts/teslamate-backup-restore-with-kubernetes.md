---
title: "Teslamate Backup & Restore With Kubernetes"
date: 2022-10-18T17:11:29Z
draft: false
---

## Overview

This document serves as an augmentation with the official Teslamate [backup and restore](https://docs.teslamate.org/docs/maintenance/backup_restore) documentation for users who are leveraging Kubernetes instead of `docker` or `docker-compose` for their installations.

## Get pod name for `teslamate-db`

> NOTE: replace "teslamate-db" if your db pod is named differently

``` shell
root@k8s04:~/homelab/k3s# PODNAME=$(kubectl get pods -n teslamate | grep teslamate-db | cut -d ' ' -f1)
echo $PODNAME
```

## Create backup file teslamate.bck

``` shell
root@k8s04:~/homelab/k3s# kubectl exec --stdin --tty $PODNAME -n teslamate -- pg_dump -U teslamate teslamate > /var/lib/postgresql/data/teslamate.bck
```

> NOTE: Change `/var/lib/postgresql/data/teslamate.bck` to a location that will be accessible to the new DB container. For most users (especially those following the [Docker Install](https://docs.teslamate.org/docs/installation/docker) documentation), this will be in the /var/lib/postgresql/data folder, mapped to some host or NAS storage.

## Scale teslamate to 0 replicas (effectively stopping the Teslamate container)

``` shell
root@k8s04:~/homelab/k3s# kubectl scale --replicas=0 -n teslamate deployment.apps/teslamate
```

> NOTE: Replace `deployment.apps/teslamate` with your deployment name, if different

## `kubectl exec` into the Postgres DB container

``` shell
root@k8s04:~/homelab/k3s# kubectl exec --stdin --tty $PODNAME -n teslamate -- /bin/bash
```

## (Inside the container) Drop existing data and reinitialize

``` shell
I have no name!@teslamate-db-56549465bf-bx5tq:/$ psql -U teslamate << .
drop schema public cascade;
create schema public;
create extension cube;
create extension earthdistance;
CREATE OR REPLACE FUNCTION public.ll_to_earth(float8, float8)
    RETURNS public.earth
    LANGUAGE SQL
    IMMUTABLE STRICT
    PARALLEL SAFE
    AS 'SELECT public.cube(public.cube(public.cube(public.earth()*cos(radians(\$1))*cos(radians(\$2))),public.earth()*cos(radians(\$1))*sin(radians(\$2))),public.earth()*sin(radians(\$1)))::public.earth';
.
```

## (Inside the container) Restore Teslamate Data

``` shell
I have no name!@teslamate-db-56549465bf-bx5tq:/$ psql -U teslamate -d teslamate < /var/lib/postgresql/data/teslamate.bck
```

## Exit the container back to the host

``` shell
I have no name!@teslamate-db-56549465bf-bx5tq:/$ exit
exit
root@k8s04:~/homelab/k3s# 
```

## Scale teslamate to 1 replicas (effectively starting the Teslamate container)

``` shell
root@k8s04:~/homelab/k3s# kubectl scale --replicas=0 -n teslamate deployment.apps/teslamate
```

> NOTE: Replace `deployment.apps/teslamate` with your deployment name, if different
