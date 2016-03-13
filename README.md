# Scalable and resilient Django with Kubernetes

This repository contains code and notes to get a sample Django
application running on a Kubernetes cluster. It is meant to go along
with a [blog post describing this in some
detail](https://harishnarayanan.org/writing/kubernetes-django/).

## Preliminary steps

1. [Install Docker](https://docs.docker.com/engine/installation/).

2. Take a look at and get a feel for the [example
application](https://github.com/hnarayanan/kubernetes-django/tree/master/containers/app)
used in this repository. It is a simple blog application built by
following the excellent [Django Girls
Tutorial](http://tutorial.djangogirls.org).

3. [Setup a cluster managed by
Kubernetes](http://kubernetes.io/docs/getting-started-guides/). The
effort required to do this can be substantial, so one easy way to get
started is to sign up (for free) on Google Cloud Platform and use a
managed version of Kubernetes called [Google Container
Engine](https://cloud.google.com/container-engine/) (GKE).

   1. Create an account on Google Cloud Platform and update your
      billing information.

   2. Install the [command line
      interface](https://cloud.google.com/sdk/).

   3. Create a project (that we'll call `$GCP_PROJECT`) using the web
      interface.

   4. Now, we're ready to set some basic configuration.

      ````
      gcloud config set project $GCP_PROJECT
      gcloud config set compute/zone us-central1-b
      ````

   5. Then we create the cluster itself.

      ````
      gcloud container clusters create demo
      gcloud container clusters list
      ````

   6. Finally, we configure `kubectl` to talk to the cluster.

      ````
      gcloud container clusters get-credentials demo
      kubectl get nodes
      ````

4. (WIP!) Setup a persistent store for the database. In this example we're
going to be using Persistent Disks from Google Cloud Platform. In
order to make one of these, we create a disk and format it (using an
instance that's temporarily created just for this purpose).

````
gcloud compute disks create pg-data-disk --size 50GB
gcloud compute instances create pg-disk-formatter
gcloud compute instances attach-disk pg-disk-formatter --disk pg-data-disk
gcloud compute config-ssh
ssh pg-disk-formatter.$GCP_PROJECT
    sudo mkfs.ext4 -F /dev/sdb
    exit
gcloud compute instances detach-disk pg-disk-formatter --disk pg-data-disk
gcloud compute instances delete pg-disk-formatter
````

## Create and publish Docker containers

For this project, we'll be using [Docker Hub](https://hub.docker.com/)
to host and deliver our containers. If you're interested in a private
repository, you need to instead use something like [Google Container
Registry](https://cloud.google.com/container-registry/).

### PostgreSQL

Build the container:

````
cd containers/database
docker build -t hnarayanan/postgresql:9.5 .
````

You can check it out locally if you want:

````
docker run --name database -e POSTGRES_DB=app_db -e POSTGRES_PASSWORD=app_db_pw -e POSTGRES_USER=app_db_user -d hnarayanan/postgresql:9.5
# Echoes $PROCESS_ID to the screen
docker exec -i -t $PROCESS_ID bash
````

Push it to a repository:

````
docker login
docker push hnarayanan/postgresql:9.5
````

### Django app running within Gunicorn

Build the container (TODO: Split into SQLite3 and PostgreSQL versions):

````
cd containers/app
docker build -t hnarayanan/djangogirls-app:0.8 .
````

You can check it out locally if you want:

````
# docker run --name some-app --link some-postgres:postgres -d application-that-uses-postgres
````

Push it to a repository:

````
docker push hnarayanan/djangogirls-app:0.8
````

## Deploy these containers to the Kubernetes cluster

### Django app running within Gunicorn (first with SQLite3)

````
kubectl create -f kubernetes/app/replication-controller-sqlite3.yaml
kubectl create -f kubernetes/app/service.yaml

kubectl get pods
kubectl get svc

kubectl scale rc app-sqlite3 --replicas=3
kubectl get pods

kubectl describe pod <pod-id>
kubectl logs <pod-id>
````

You can check resiliency by deleting one or more app pods and see it
respawn.

````
kubectl delete pod <pod-id>
kubectl get pods
````

### PostgreSQL

Even though our application only requires a single PostgreSQL instance
running, we still run it under a (pod) replication controller. This
way, we have a service that monitors our database pod and ensures that
one instance is running even if something weird happens, such as the
underlying node fails.

````
kubectl create -f kubernetes/database/replication-controller.yaml
kubectl get replicationcontrollers
kubectl get pods
kubectl describe pod <pod-id>
kubectl logs <pod-id>

kubectl create -f kubernetes/database/service.yaml
kubectl get services
kubectl describe services database
````

### Django app running within Gunicorn (now, with PostgreSQL)

````
kubectl create -f kubernetes/app/replication-controller-postgres.yaml

kubectl get pods
kubectl get svc
````

Setup initial migrations and create an initial user
````
kubectl exec <some-app-postgres-pod-id> -- python /app/manage.py migrate
kubectl exec -it <some-app-postgres-pod-id> -- python /app/manage.py createsuperuser
````

Scale the PostgreSQL pods to 3 replicas, and remove all SQLite3 pods

````
kubectl scale rc app-sqlite3 --replicas=0
kubectl get pods

kubectl scale rc app-postgres --replicas=3
kubectl get pods
````

## Static Files

````
gsutil mb gs://django-kubernetes-assets
gsutil defacl set public-read gs://django-kubernetes-assets
cd django-k8s/containers/app
./manage.py collectstatic --noinput
gsutil -m cp -r static/* gs://django-kubernetes-assets
````

## TODO: Unmerged notes

````
- Monitoring UI

- Secrets Resource
  echo mysecretpassword | base64
  <paste into secrets file>
  kubectl create -f kubernetes_configs/db_password.yaml

- PostgreSQL Persistent Volume (Claims)
  kubectl create -f kubernetes/database/persistent-volume.yaml
  kubectl get pv
  kubectl create -f kubernetes/database/persistent-volume-claim.yaml
  kubectl get pvc
````
