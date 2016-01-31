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
