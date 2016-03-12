## Containers

1. PostgreSQL

Build the container:

````
cd containers/postgresql
docker build -t hnarayanan/postgresql:9.5 .
````

"Test" the container:

````
docker run --name database -e POSTGRES_DB=app_db -e POSTGRES_PASSWORD=app_db_pw -e POSTGRES_USER=app_db_user -d postgresql
# docker run -it --link database:postgres --rm postgres sh -c 'exec psql -h "$POSTGRES_PORT_5432_TCP_ADDR" -p "$POSTGRES_PORT_5432_TCP_PORT" -U app_db_user'

````

Push it to a repository:

For this project, we'll be using [Docker Hub](https://hub.docker.com/)
to host and deliver our containers. If you're interested in a private
repository, you need to instead use something like [Google Container
Registry](https://cloud.google.com/container-registry/).

````
docker login
docker push hnarayanan/postgresql:9.5
````

2. Django + uWSGI

https://github.com/mbentley/docker-django-uwsgi-nginx/blob/master/Dockerfile
http://michal.karzynski.pl/blog/2015/04/19/packaging-django-applications-as-docker-container-images/

````
# docker run --name some-app --link some-postgres:postgres -d application-that-uses-postgres
````

3. NGINX

````
cd containers/nginx
docker build -t hnarayanan/nginx:1.9.11 .

(docker run --name nginx -d hnarayanan/nginx:1.9.11)
docker push hnarayanan/nginx:1.9.11

````

## Infrastructure setup

1. A Kubernetes cluster using Google Container Engine (GKE)

   ````
   gcloud config set project $GCP_PROJECT
   gcloud config set compute/zone us-central1-b
   gcloud container clusters create django-k8s
   ````

2. A persistent disk for PostgreSQL

   Create a disk and format it (using an instance that's temporarily
   created just for this purpose).

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

   Setup this disk as something that's usable in Kubernetes.

   ````
   kubectl create -f kubernetes/database/persistent-volume.yaml
   kubectl get pv
   kubectl create -f kubernetes/database/persistent-volume-claim.yaml
   kubectl get pvc
   ````

## Replication Controllers

1. PostgreSQL

Even though our application only requires a single PostgreSQL instance
running, we still run it under a (pod) replication controller. This
way, we have a service that monitors our database pod and ensures that
one instance is running even if something weird happens, such as the
underlying node fails.

1. PostgreSQL

````
kubectl create -f kubernetes/database/replication-controller.yaml
kubectl get replicationcontrollers
kubectl get pods

kubectl stop -f kubernetes/database/replication-controller.yaml
kubectl get replicationcontrollers
kubectl get pods
````

## Services

````
kubectl create -f kubernetes/database/service.yaml
kubectl get services
kubectl describe services database

kubectl stop -f kubernetes/database/service.yaml
kubectl get services
````



docker build -t hnarayanan/djangogirls-app:0.1 .
docker push hnarayanan/djangogirls-app:0.1

kubectl create -f replication-controller.yaml
kubectl create -f service.yaml
