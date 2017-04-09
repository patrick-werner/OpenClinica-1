OpenClinica 3.4.1 via Docker
============================

This folder contains the *Dockerfile*, a startup script and the following instructions for running a Docker container  which you can use to give OpenClinica a try. An image built with this Dockerfile is available on [Docker Hub](https://registry.hub.docker.com/u/piegsaj/openclinica/).

> **IMPORTANT:** *This image is meant for trying out OpenClinica and not meant for running a production server or for storing important study data.*

Setup
-----

### 1. Install Docker

* follow the [installation instructions](http://docs.docker.com/installation/) for your host system
    * **note:** the maximum RAM size can be adjusted through the user interface of VirtualBox (run it from the start menu, stop the virtual machine, change the configuration to e.g. 4096MB, close it and start the virtual machine using `docker-machine`)

### 2. Start a data-only container with a volume for the database

```sh
sudo docker run --name=ocdb-data -v /var/lib/postgresql/data postgres:8 true
```

### 3. Start the PostgreSQL database server

```sh
sudo docker run --name=ocdb -d --volumes-from ocdb-data -e POSTGRES_PASSWORD=postgres123 -p 5432:5432 postgres:8
```
*Remove `-p 5432:5432` here, if the database port should not be published.*

### 4. Initialize the database

```sh
sudo docker exec ocdb su postgres -c $'psql -c "CREATE ROLE clinica LOGIN ENCRYPTED PASSWORD \'clinica\' SUPERUSER NOINHERIT NOCREATEDB NOCREATEROLE" && psql -c "CREATE DATABASE openclinica WITH ENCODING=\'UTF8\' OWNER=clinica" && echo "host all  clinica    0.0.0.0/0  md5" >> $PGDATA/pg_hba.conf && /usr/lib/postgresql/$PG_MAJOR/bin/pg_ctl reload -D $PGDATA'
```

### 5. Start a data-only container with a volume for files written by OpenClinica

```sh
sudo docker run --name=oc-data -v /tomcat/openclinica.data piegsaj/openclinica true
```

### 6. Start Tomcat serving OpenClinica and OpenClinica-ws

```sh
sudo docker run --name=oc -h oc -d --volumes-from oc-data -p 80:8080 -e TOMCAT_PASS="admin" -e LOG_LEVEL=INFO -e TZ=UTC-1 --link=ocdb:ocdb piegsaj/openclinica
```
*The environment variables for log level and timezone are optional here.*

### 7. Get the external IP address

If you are using boot2docker simply call from your host system:

```sh
boot2docker ip
```

### 8. Run OpenClinica

* open up [http://&lt;ip.of.your.host&gt;/OpenClinica](http://<ip.of.your.host>/OpenClinica) in your browser
* first time login credentials: `root` / `12345678`

Operation
---------

**To show the OpenClinica logs:**

```sh
sudo docker logs -f oc
```

**To backup a database dump to the current directory on the host:**

```
echo "postgres123" | sudo docker run -i --rm --link ocdb:ocdb -v $PWD:/tmp postgres:8 sh -c 'pg_dump -h ocdb -p $OCDB_PORT_5432_TCP_PORT -U postgres -F tar -v openclinica > /tmp/ocdb_pg_dump_$(date +%Y-%m-%d_%H-%M-%S).tar'
```

**To backup the OpenClinica data folder to the current directory on the host:**

```sh
sudo docker run --rm --volumes-from oc-data -v $PWD:/tmp piegsaj/openclinica tar cvf /tmp/oc_data_backup_$(date +%Y-%m-%d_%H-%M-%S).tar /tomcat/openclinica.data
```

**To remove all stopped containers on the host, except those named *-data:**

```
docker ps -a -f status=exited | grep -v '\-data *$'| awk '{if(NR>1) print $1}' | xargs -r docker rm
```

Contribute
----------

Feedback is welcome. The source is available on [Github](https://github.com/JensPiegsa/OpenClinica/). Please [report any issues](https://github.com/JensPiegsa/OpenClinica/issues).

