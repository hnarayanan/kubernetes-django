This repository contains code and notes to get a Django application
working on Google Container Engine with Kubernetes.

Outline:

- Install Docker

  - The toolbox if you're on Mac or Windows

    https://www.docker.com/toolbox

  - Classic instructions if you're on Linux

    https://foo

- The example Django application is the Django girls blog tutorial

- Containerize it initially with https://hub.docker.com/r/library/django/

  ````
  cd djangogirls
  docker build -t djangogirls-app .

  docker run --name djangogirls-app-copy-3 -p 8000:8000 -d djangogirls-app
  docker-machine ip default
  http://192.168.99.100:8000
  ````

move sqlite3 db out of the file system
run migrations etc. to setup (initial) state

docker run --name database -e POSTGRES_USER=db_user -e POSTGRES_PASSWORD=db_password -d postgres

- Greatly improve this toy setup

  - Postgres backend

    https://medium.com/google-cloud/basic-rails-app-with-docker-a08eba3c2197





django(-uwsgi)
postgresql
...

each component gets a folder with a Dockerfile + supporting files
