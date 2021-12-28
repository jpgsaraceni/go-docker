# Go app Docker Containers

This repo contains code written following the tutorial on [Docker Docs](https://docs.docker.com/language/golang/). The details of Docker and Docker Compose concepts I learned are in [another repo](https://github.com/jpgsaraceni/docker-first-steps). My notes here are complementary and/or refer to Go specific conteiners.

All commands here are run on Linux, but they are the same for windows (except for the sudo at the beginning). It's also possible to run docker without sudo, there's some documentation for that somewhere.

## Example app

The application here was not written by me. It is part of the tutorial and is available on [another GitHub repo](https://github.com/olliefr/docker-gs-ping), so I kept it out of my repo to focus on what I'm actually learning here - Docker.

## Dockerfile

This file must be in the root of the project. It tells Docker how to build the image.

The (optional) first line of the Dockerfile is the `#syntax` parsing directive, which tells Docker what syntax to use while parsing the Dockerfile.

The next line is the `FROM` command. This gives a base image to build the image from.

The `WORKDIR` indicates the path in the image to use as the default destination for all commands.

The `COPY` line takes two parameter: what to copy and where to. The first parameter is relative to the directory where Dockerfile is, and the second to WORKDIR.

`ENV` can optionally be used to set environment variables for the dockerised app.

The `RUN` command executes when building the image.

`CMD` tells Docker to execute the command when a container is started from the image built.

## Image

To build the local image:

```shell
sudo docker build --tag docker-gs-ping .
```

To list local images

```shell
sudo docker image ls
# or shor
sudo docker images
```

Add another tag to an image by running:

```shell
sudo docker image tag <IMAGE_NAME>:<EXISTENT_TAG> <IMAGE_NAME>:<NEW_TAG>
```

If you run `docker images` again you will see two entries on the list with the same image id but different tags.

To remove an image or tag run `docker image rm <IMAGE_ID OR IMAGE_NAME:TAG>`

## Multistage builds

These will build much more compact images, by "rebuilding" the image with only the essentials. This tutorial doesn't cover in depth, so I'm just copying the Dockerfile.multistage file from there, and will later take a look at a [specific guide](https://docs.docker.com/develop/develop-images/multistage-build/) on Docker docs.

To build from this new Dockerfile (that is not the default):

```shell
sudo docker build -t docker-gs-ping:multistage -f Dockerfile.multistage .
```

## Run the container

```shell
sudo docker run docker-gs-ping
```

Will give you a very useless container running, because it's isolated and won't receive requests from outside it's network. To expose it, use the `--publish` or `-p` flag, followed by `<HOST_PORT>:<CONTAINER_PORT>`. For example, to expose the container's port 8080, which it listens to, on port 3000 of the host (`-d` flag runs the container in the background):

```shell
sudo docker run -dp 3000:8080 docker-gs-ping:multistage
```

## Network

Networks allow containers to communicate. To create one in the background called "mynet":

```shell
sudo docker network create -d bridge mynet
```

Bridge is one possible network configuration. You can list active networks:

```shell
sudo docker network list
```

## Persisting Data

You can create a volume to persist your data (this one will be named "roach" as the tutorial runs an image of CockroachDB):

```shell
sudo docker volume create roach
```

To pull an image from Docker Hub and run locally:

```shell
sudo docker run -d \
  --name roach \
  --hostname db \
  --network mynet \
  -p 26257:26257 \
  -p 8080:8080 \
  -v roach:/cockroach/cockroach-data \
  cockroachdb/cockroach:latest-v20.1 start-single-node \
  --insecure
```

## Configure DB engine

To start SQL shell in same container CockroachDB is running:

```shell
sudo docker exec -it roach ./cockroach sql --insecure
```

In SQL shell, create a DB, user and grant permissions. Then exit the SQL shell:

```sql
CREATE DATABASE mydb;
CREATE USER totoro;
GRANT ALL ON DATABASE mydb TO totoro;
quit
```

## Extended example

```shell
sudo docker build --tag docker-gs-ping-roach .
```

Run the image, setting the env variables (the DB is running in insecure mode, so the password can be anything):

```shell
sudo  docker run -it --rm -d \
  --network mynet \
  --name rest-server \
  -p 80:8080 \
  -e PGUSER=totoro \
  -e PGPASSWORD=myfriend \
  -e PGHOST=db \
  -e PGPORT=26257 \
  -e PGDATABASE=mydb \
  docker-gs-ping-roach
```

`rest-server` is how you will refer to this container in docker commands, to stop, run, remove.

`db` is how you will refer in your app.

### Using the app

Your container is now running. To send a `GET` request to `/` (message counter):

```shell
curl localhost
```

To send a `POST` request to `/send` (send message):

```shell
curl --request POST \
  --url http://localhost/send \
  --header 'content-type: application/json' \
  --data '{"value": "Hello, Docker!"}'
```

### Verify persistence

Stop the server and DB containers:

```shell
sudo docker container stop rest-server roach
```

Remove:

```shell
sudo docker container rm rest-server roach
```

Start DB then server again:

```shell
sudo  docker run -d \
  --name roach \
  --hostname db \
  --network mynet \
  -p 26257:26257 \
  -p 8080:8080 \
  -v roach:/cockroach/cockroach-data \
  cockroachdb/cockroach:latest-v20.1 start-single-node \
  --insecure
```

```shell
sudo docker run -it --rm -d \
  --network mynet \
  --name rest-server \
  -p 80:8080 \
  -e PGUSER=totoro \
  -e PGPASSWORD=myfriend \
  -e PGHOST=db \
  -e PGPORT=26257 \
  -e PGDATABASE=mydb \
  docker-gs-ping-roach
```

See that the messages are still there on localhost `/` route.

Remove the containers again to continue.

## Docker compose

Makes life a lot easier, as all Docker Commands will be stored in one file, and all you have to do to run your containers is run `docker-compose.yml` file.

Check [docker compose documentation](https://docs.docker.com/compose/) for more details.

Docker Compose automatically reads .env variables. Set any PGPASSWORD variable in it for docker compose to use.

Variable substitution, like `PGUSER=${PGUSER:-totoro}` allows to set a default value to an env variable if none are found in the .env file.

To check for errors in your yml file:

```shell
docker-compose config
```

To build and run the app:

```shell
sudo docker-compose up --build
```

This will throw an error because the user fails authenticatins. To correct this, keep the coker compose running and open a new terminal to run the SQL shell like before:

```shell
sudo docker exec -it roach ./cockroach sql --insecure
```

And repeat the steps to create a user.

To remove the containers that are running your server:

```shell
sudo docker-compose down
```

To run in detached mode:

```shell
sudo docker-compose up --build -d
```

And to stop:

```shell
sudo docker-compose stop
```
