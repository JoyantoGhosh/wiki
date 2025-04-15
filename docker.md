# Docker

## Table of content
1. [Cheatsheet](#cheatsheet)
2. [Docker Image](#dockerimages)
3. [Docker Compose](#dockercompose)
4. [Docker Registry](#dockerregistry)
5. [Installation](#installation)


## Concept
- Unlike VMs, containers are meant to run a specific task or process.
- Container exists until the process inside it is alive.

-----

## Cheatsheet
```
$ docker run nginx                                      // This pulls the image with tag 'latest'
$ docker run nginx:1.26                                 // This pulls the image with tag '1.26'

$ docker run -i jglab/testapp                           // This is to have input prompt in the docker container
$ docker run -it jglab/testapp                          // This is to have a pseudo terminal attached to the container to see the prompts

$ docker run -p 8080:5000 jglab/testapp                 // This maps host port to the container port
$ docker run -v /opt/sqlbackup:/var/lib/mysql mysql     // This maps host directory to the container directory

$ docker run -d nginx                                   // Detach the container
$ docker attach <container name/id>                     // Attach the Detached container
$ docker exec nginx cat /etc/hosts

                                                        // To define the dependencies
$ docker run -d --name=redis redis
$ docker run -d --name=webapp  -p 8080:5000 --link redis:redis jglab/webapp

$ docker inspect <container name/id>                    // To get details of the container 
$ docker logs <container name/id>                       // To get logs of the StdOut in the container

$ docker ps -a
$ docker stop <container name/id>
$ docker remove <container name/id>

$ docker pull
$ docker image list
$ docker remove images <image name/id>

$ docker run -e DEBUG_ENABLE=true jglab/mypy

```
-----

## DockerImages

### How to create a docker image?
Below mentioned are the generic steps we perform to run any application.
1. Have a base image
2. Update packages of the base image
3. Install application OS level dependencies
4. Install application package level dependencies
5. Copy the source code
6. Run the application

The same sequence has to be followed and structured in a `Dockerfile`.
```
FROM ubuntu
RUN apt update && apt upgrade
RUN apt install python -y
COPY . /opt/src
CMD python main.py
```

**Note : -** 
- Every `Dockerfile` should start with a __FROM__ instruction.
- Each instruction creates a layer in the docker image

#### When to use CMD and ENTRYPOINT?

- **CMD** is used when application command is defined and no argument need to be passed by the user

```
FROM Ubuntu
CMD sleep 10
```

`$ docker run ubuntu-sleep`

- **ENTRYPOINT** is used when the application command needs to be parameterised.

```
FROM Ubuntu
ENTRYPOINT sleep
```

`$ docker run ubuntu-sleep 10`

**Note: -** 
- This will throw an exception if no input is passed during container run.
- `$ docker run ubuntu-sleep`
- To pass a default value, use both ENTRYPOINT and CMD

```
FROM Ubuntu
ENTRYPOINT ["sleep"]
CMD ["10"]
```

`$ docker run ubuntu-sleep`

#### How to create an image locally?
```
$ docker build .
or
$ docker build . -f Dockerfile -t jglab/mypy
```

#### How to push an image to the docker registry?
```
$ docker build . -f Dockerfile -t jglab/mypy

$ docker login
$ docker push jglab/mypy
$ docker history jglab/mypy
```
-----

## DockerCompose

### Installation
`docker-compose` does not automatically get installed with Docker. It has to be installed separately. Although it is packaged with docker-desktop.

```
$ curl -SL https://github.com/docker/compose/releases/download/v2.35.0/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
$ chmod +x /usr/local/bin/docker-compose
```

Ref: https://docs.docker.com/compose/install/standalone/

### Why we need docker-compose?
The containerised application does not work stand alone, it works as a full __application stack__ which consists of the actual application, the database, the cache, the messaging, etc.

It is not feasible to bring up containers and configure them one-by-one manually for full application stack everytime.

Below mentioned are the generic steps we perform to run any application stack.
1. Run the dependent containers, example- redis and postgres.
2. Run the application services containers, example- webapp and ingestion.
3. Link the application services containers to the respective dependent containers to establish networking.

The same sequence has to be followed and structured in a `docker-compose.yaml`.

```
---
version: 1                          // Not mandatory to use in version 1
redis:
    image: redis
db:
    image: postgres
webapp:
    image: jglab/webapp:1.0
    ports:
        - 8080:5000
    links:
        - redis
ingestion:
    build: ./ingestion              // This is the location of `Dockerfile`
    links:
        - db
```
- In `version 1`, docker-compose attaches all the containers in default bridge network.
```
---
version: "2"                          // Mandatory to use from version 2 and above
services:
    redis:
        image: redis
        networks:
            - backend
    db:
        image: postgres
        environment:
            POSTGRES_USER: postgres
            POSTGRES_PASSWORD: postgres
        networks:
            - backend
    webapp:
        image: jglab/webapp:1.0
        ports:
            - 8080:5000
        depends_on:
            - redis
        networks:
            - frontend
    ingestion:
        build: ./ingestion              // This is the location of `Dockerfile`
        depends_on:
            - db
        networks:
            - backend
networks:
    frontend:
    backend:
```
- In `version 2`, docker compose creates a dedicated bridge network and then attaches all the containers to it.
- Containers now can communicate using service names and links are no more required.
- Multiple networks can be created to bifurcate the trafic.

`$ docker-compose up`

- When the `image` is specified in the file, the docker pull the image from the registry
- When the `build` is specified in the file, the docker first builds the image using Dockerfile and then use the image to spawn a container  
-----

## DockerRegistry

- Images are pushed to docker public registry `https://hub.docker.com/`
- Images can also be pushed to locally hosted docker registry
```
$ docker run -d --name=registry -p 5000:5000 registry
$ docker build . -t myWebapp
$ docker push localhost:5000/myWebapp
$ docker pull localhost:5000/myWebapp
```
**Note: -**
- The localhost can be changed to the IP address of the registry container.
-----

## Installation

### Components
- docker-ce
- docker-ce-cli
- containerd.io
- docker-buildx-plugin
- docker-compose-plugin

Check OS release name
```
$ cat /etc/*release*
    -----
    DISTRIB_ID=Ubuntu
    DISTRIB_RELEASE=24.04
    DISTRIB_CODENAME=noble
    -----
```

Remove older packages
```
$ for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

Add docker repository to apt sources
```
$ sudo apt-get update
$ sudo apt-get install ca-certificates curl
$ sudo install -m 0755 -d /etc/apt/keyrings
$ sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
$ sudo chmod a+r /etc/apt/keyrings/docker.asc

$ echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
$ sudo apt-get update
```

Install docker packages
```
$ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

$ sudo apt-get update && sudo apt upgrade

$ docker version
    -----
    Client: Docker Engine - Community
    Version:           28.0.2
    -----
```



Post installation issue & fix:
```
# Issue
$ docker ps -a
    -----
    permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock:
    -----

# Fix
$ sudo groupadd docker
$ sudo usermod -aG docker $USER
```

Ref: https://docs.docker.com/engine/install/ubuntu/
