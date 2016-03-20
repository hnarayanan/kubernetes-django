# Scalable and resilient Django with Kubernetes

This repository contains code and notes to get a sample Django
application running on a Kubernetes cluster. It is meant to go along
with a [related blog post][blog-post] that provides more context and
explains some of the theory behind the steps that follow.

## Preliminary steps

1. [Install Docker][docker-installation].

2. Take a look at and get a feel for the [example Django
application][example-app] used in this repository. It is a simple blog
that’s built following the excellent [Django Girls
Tutorial][django-girls-tutorial].

3. [Setup a cluster managed by Kubernetes][kubernetes]. The effort
required to do this can be substantial, so one easy way to get started
is to sign up (for free) on Google Cloud Platform and use a managed
version of Kubernetes called [Google Container Engine][GKE] (GKE).

   1. Create an account on Google Cloud Platform and update your
      billing information.

   2. Install the [command line interface][gcp-sdk].

   3. Create a project (that we'll refer to henceforth as
      `$GCP_PROJECT`) using the web interface.

   4. Now, we're ready to set some basic configuration.

      ````
      gcloud config set project $GCP_PROJECT
      gcloud config set compute/zone europe-west1-d
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

## Create and publish Docker containers

For this demo, we'll be using [Docker Hub](https://hub.docker.com/) to
host and deliver our containers. And since this demo doesn't contain
any secret information, these containers will be exposed to the
public.

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

Build the container:

````
cd containers/app
docker build -t hnarayanan/djangogirls-app:orange .
````

Push it to a repository:

````
docker push hnarayanan/djangogirls-app:orange
````

----------
To demonstrate rolling updates later, we make an alternative version
of this blog that uses a different header colour. For this, we modify
the source


Build

Push
----------

## Deploy these containers to the Kubernetes cluster

### PostgreSQL

Even though our application only requires a single PostgreSQL instance
running, we still run it under a (pod) replication controller. This
way, we have a service that monitors our database pod and ensures that
one instance is running even if something weird happens, such as the
underlying node fails.

````
cd  kubernetes/database
kubectl create -f replication-controller.yaml

kubectl get rc
kubectl get pods

kubectl describe pod <pod-id>
kubectl logs <pod-id>
````

Now we start a service to point to the pod.

````
cd  kubernetes/database
kubectl create -f service.yaml

kubectl get svc
kubectl describe svc database
````

### The first version of the Django app (orange) Gunicorn

We begin with three app pods (copies of the app container) talking to
the single database.

````
cd kubernetes/app
kubectl create -f replication-controller-orange.yaml
kubectl get pods

kubectl describe pod <pod-id>
kubectl logs <pod-id>
````

Then we start a service to point to the pod. This is a load-balancer
with an external IP so we can access the site.

````
cd kubernetes/app
kubectl create -f service.yaml
kubectl get svc
````

Before we access the website using the external IP presented by
`kubectl get svc`, we need to do a few things:

1. Perform initial migrations:

   ````
   kubectl exec <some-app-orange-pod-id> -- python /app/manage.py migrate
   ```

2. Create an intial user for the blog:

   ````
   kubectl exec -it <some-app-orange-pod-id> -- python /app/manage.py createsuperuser
   ````

3. Have a CDN host static files since we don't want to use Gunicorn
   for this. This demo uses Google Cloud storage, but you're free to
   use whatever you want. Just make sure `STATIC_URL` in
   `containers/app/mysite/settings.py` reflects where the files are.

   ````
   gsutil mb gs://demo-assets
   gsutil defacl set public-read gs://demo-assets
   cd django-k8s/containers/app
   ./manage.py collectstatic --noinput
   gsutil -m cp -r static/* gs://demo-assets
   ````

## Play around to understand Kubernetes' API

Now, suppose your site isn't getting much traffic, you can gracefully
scale down the number of running pods to 1. (Similarly you can
increase the number of pods if your traffic is starting to grow!)

````
kubectl scale rc app-sqlite3 --replicas=1
kubectl get pods
````

You can check resiliency by deleting one or more app pods and see it
respawn.

````
kubectl delete pod <pod-id>
kubectl get pods
````

### Django app running within Gunicorn (now, with PostgreSQL)

````
kubectl create -f kubernetes/app/replication-controller-postgres.yaml

kubectl get pods
kubectl get svc
````

Scale the PostgreSQL pods to 3 replicas, and remove all SQLite3 pods

````
kubectl scale rc app-sqlite3 --replicas=0
kubectl get pods

kubectl scale rc app-postgres --replicas=3
kubectl get pods
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

[blog-post]: https://harishnarayanan.org/writing/kubernetes-django/
[docker-installation]: https://docs.docker.com/engine/installation/
[example-app]: https://github.com/hnarayanan/kubernetes-django/tree/master/containers/app
[django-girls-tutorial]: http://tutorial.djangogirls.org
[kubernetes]: http://kubernetes.io/docs/getting-started-guides/
[GKE]: https://cloud.google.com/container-engine/
[gcp-sdk]: https://cloud.google.com/sdk/
