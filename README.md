# Scalable and resilient Django with Kubernetes

This repository contains code and notes to get a sample Django
application running on a Kubernetes cluster. It is meant to go along
with a [related blog post][blog-post] that provides more context and
explains some of the theory behind the steps that follow.

## Preliminary steps

1. Fetch the source code for this example.
   ````
   git clone https://github.com/hnarayanan/kubernetes-django.git
   ````

2. [Install Docker][docker-install].

3. Take a look at and get a feel for the [example Django
application][example-app] used in this repository. It is a simple blog
that’s built following the excellent [Django Girls
Tutorial][django-girls-tutorial].

4. [Setup a cluster managed by Kubernetes][kubernetes-install]. The
effort required to do this can be substantial, so one easy way to get
started is to sign up (for free) on Google Cloud Platform and use a
managed version of Kubernetes called [Google Container Engine][GKE]
(GKE).

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

For this example, we'll be using [Docker Hub](https://hub.docker.com/)
to host and deliver our containers. And since we're not working with
any sensitive information, we'll expose these containers to the
public.

### PostgreSQL

Build the container, remembering to use your own username on Docker
Hub instead of `hnarayanan`:

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
docker build -t hnarayanan/djangogirls-app:1.2-orange .
````

Push it to a repository:

````
docker push hnarayanan/djangogirls-app:1.2-orange
````

We're going to see how to perform rolling updates later in this
example. For this, let's create an alternative version of our app that
simply has a different header colour, build a new container app and
push that too to the container repository.

````
cd containers/app
emacs blog/templates/blog/base.html

# Add the following just before the closing </head> tag
    <style>
      .page-header {
        background-color: #ac4142;
      }
    </style>

docker build -t hnarayanan/djangogirls-app:1.2-maroon .
docker push hnarayanan/djangogirls-app:1.2-maroon
````

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

### Django app running within Gunicorn

We begin with three app pods (copies of the orange app container)
talking to the single database.

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
   ````

2. Create an intial user for the blog:
   ````
   kubectl exec -it <some-app-orange-pod-id> -- python /app/manage.py createsuperuser
   ````

3. Have a CDN host static files since we don't want to use Gunicorn
   for serving these. This demo uses Google Cloud storage, but you're
   free to use whatever you want. Just make sure `STATIC_URL` in
   `containers/app/mysite/settings.py` reflects where the files are.
   ````
   gsutil mb gs://demo-assets
   gsutil defacl set public-read gs://demo-assets

   cd django-k8s/containers/app
   virtualenv --distribute --no-site-packages venv
   source venv/bin/activate
   pip install Django==1.9.5
   export DATABASE_ENGINE='django.db.backends.sqlite3'
   ./manage.py collectstatic --noinput
   gsutil -m cp -r static/* gs://demo-assets
   ````

At this point you should be able to load up the website by visiting
the external IP for the app service (obtained by running `kubectl get
svc`) in your browser.

Go to `http://app-service-external-ip/admin/` to login using the
credentials you setup earlier (while creating a super user), and
return to the site to add some blog posts. Notice that as you refresh
the site, the name of the app pod serving the site changes, while the
content stays the same.

## Play around to get a feeling for Kubernetes' API

Now, suppose your site isn't getting much traffic, you can gracefully
*scale* down the number of running application pods to one. (Similarly
you can increase the number of pods if your traffic starts to grow!)

````
kubectl scale rc app-orange --replicas=1
kubectl get pods
````

You can check *resiliency* by deleting one or more app pods and see it
respawn.

````
kubectl delete pod <pod-id>
kubectl get pods
````

Notice Kubernetes will spin up the appropriate number of pods to match
the last known state of the replication controller.

Finally, to show how we can migrate from one version of the site to
the next, we'll move from the existing orange version of the
application to another version that's maroon.

First we scale down the orange version to just one copy:

````
kubectl scale rc app-orange --replicas=1
kubectl get pods
````

Then we spin up some copies of the new maroon version:

````
cd kubernetes/app
kubectl create -f replication-controller-maroon.yaml
kubectl get pods
````

Notice that because the app service is pointing simply to the label
`name: app`, both the one orange and the three maroon apps respond to
http requests to the external IP.

When you're happy that the maroon version is working, you can spin
down all remaining orange versions, and delete its replication
controller.

````
kubectl scale rc app-orange --replicas=0
kubectl delete rc app-orange
````

## Cleaning up

After you're done playing around with this example, remember to
cleanly discard the compute resources we spun up for it.

````
gcloud container clusters delete demo
gsutil -m rm -r gs://demo-assets
````

## And coming in the future

Future iterations of this demo will have additional enhancements, such
as using a Persistent Volume for PostgreSQL data and learning to use
Kubernetes' Secrets API to handle secret passwords. Keep an eye on
[the issues for this project][issues] to find out more. And you're
free to help out too!

[blog-post]: https://harishnarayanan.org/writing/kubernetes-django/
[docker-install]: https://docs.docker.com/engine/installation/
[kubernetes-install]: http://kubernetes.io/docs/getting-started-guides/
[example-app]: https://github.com/hnarayanan/kubernetes-django/tree/master/containers/app
[django-girls-tutorial]: http://tutorial.djangogirls.org
[GKE]: https://cloud.google.com/container-engine/
[gcp-sdk]: https://cloud.google.com/sdk/
[issues]: https://github.com/hnarayanan/kubernetes-django/issues
