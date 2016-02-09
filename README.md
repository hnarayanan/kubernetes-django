## Resources

1. A persistent disk for PostgreSQL

Create a disk and format it (using an instance that's temporarily
created just for this purpose).

````
gcloud compute disks create pg-data-disk --size 50GB
gcloud compute instances create pg-disk-formatter
gcloud compute instances attach-disk pg-disk-formatter --disk pg-data-disk
ssh pg-disk-formatter.$GCP_PROJECT
    sudo mkfs.ext4 -F /dev/sdb
    exit
gcloud compute instances detach-disk pg-disk-formatter --disk pg-data-disk
gcloud compute instances delete pg-disk-formatter
````

## Containers

1. PostgreSQL

Build the container:

````
cd containers/postgresql
docker build -t postgresql .
````

Test the container:

````
docker run --name database -e POSTGRES_DB=app_db -e POSTGRES_PASSWORD=app_db_pw -e POSTGRES_USER=app_db_user -d postgresql
# docker run -it --link database:postgres --rm postgres sh -c 'exec psql -h "$POSTGRES_PORT_5432_TCP_ADDR" -p "$POSTGRES_PORT_5432_TCP_PORT" -U app_db_user'

````

2. Django + uWSGI

````
# docker run --name some-app --link some-postgres:postgres -d application-that-uses-postgres
````

3. NGINX

## Pods

## Replication Controllers

## Services
