# Scalable and resilient Django with Kubernetes

This repository contains code and notes to get a sample Django
application working on a Kubernetes cluster running on Google Cloud
Platform. It is meant to go along with a [blog post describing this in
some detail](https://harishnarayanan.org/writing/kubernetes-django/).

## The steps

1. [Install Docker](https://docs.docker.com/engine/installation/).
2. Work through the example Django application covered in the [Django
Girls Tutorial](http://tutorial.djangogirls.org).
3. Create a Kubernetes cluster. One easy way to do this is to sign up
(for free) on Google Cloud Platform and set up a Google Container
Engine (GKE) cluster.
   ````
	- Set the project
		gcloud config set project django-kubernetes
	- Setup the cluster
		gcloud container clusters create demo
		gcloud container clusters list
	- Make sure kubectl is configured to see it
		gcloud container clusters get-credentials demo
		kubectl get nodes
   ````

2. Create and publish Docker containers for the components of your application


3. Deploy these to the Kubernetes cluster in stages

	- Secrets Resource
		echo mysecretpassword | base64
		<paste into secrets file>
		kubectl create -f kubernetes_configs/db_password.yaml
	- Redis
		kubectl create -f kubernetes_configs/redis_cluster.yaml
	- PostgreSQL
		Disk
			gcloud compute disks create pg-data  --size 500GB
		Build Container (Incorporating secrets?)
			docker build -t gcr.io/edgefolio-development/postgres-pw .
			gcloud docker push gcr.io/edgefolio-development/postgres-pw
		Deploy on k8s
			kubectl create -f kubernetes_config/postgres.yaml
	- Frontend
		Build Docker containers
			docker build -t gcr.io/edgefolio-development/guestbook .
			gcloud docker push gcr.io/edgefolio-development/guestbook
		Deploy on k8s
			kubectl create -f kubernetes_configs/frontend.yaml  
			kubectl get pods
			kubectl describe pod <pod-id>
			kubectl logs <pod-id>

4. Demo
   - scaling
   - deleting one pod
   - different versions, split by colour?
   - monitoring UI


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


## Static Files

gsutil mb gs://django-kubernetes-assets
gsutil defacl set public-read gs://django-kubernetes-assets
cd django-k8s/containers/app
./manage.py collectstatic --noinput
gsutil -m cp -r static/* gs://django-kubernetes-assets
