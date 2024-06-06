# Container fundamentals

There are a couple of topics that you can learn that cover most of the usecases where you need to use containers, either with only Docker or k8s:

- Architecture of containers
- How a process works in Linux and how to manage them
- Run containers, configure:
  - Entrypoint or command
  - Environment variables
  - Ports
- Build and pull container images
- Mount volumes and files
- Network

> In Linux, everything is a file!

## Container registries

- https://hub.docker.com/ (default container registry)
- https://quay.io/
- https://github.com/features/packages
- https://aws.amazon.com/ecr/

## Run a container

We will start by pulling the nginx image. You will see that docker pulls all the layers of that image.

```bash
# Check the current images
docker images

# Pull the nginx image
docker pull nginx

# Check information about the nginx image
docker images nginx
```

Once the image is pulled, then it is available in our local filesystem. Now Docker can make use of it.

Tun run a container with nginx we can use the following commmand:

```bash
# Run the nginx image
docker run -p 80 nginx
```

We can check the running containers by executing the following command in another tab or terminal:

```bash
docker ps
```

In Linux, applications use by default two output streams, STDOUT and STDERR. The logs command can show you both streams.

```bash
# In another tab/terminal check the running containers
docker logs CONTAINER_ID
```

We can stop a running container with the stop command

```bash
docker stop CONTAINER_ID
```

```bash
# List all containers (also stopped)
docker ps -a
```

If we want to restart the container we can make use of the start command

```bash
# Remove the container
docker start CONTAINER_ID
```

Finally, we can remove the container with the rm container. But before we need to stop it

```bash
# Remove the container
docker stop CONTAINER_ID
docker rm CONTAINER_ID
```


## Container images part 1

In our first example we want to build an image with a node application.

We create the source code with the following contents:

```bash
# Create a dir in ~/docker/build-image and acces it
mkdir -p ~/docker/build-image-node
cd $_

# Create the files:
# - package.json
# - server.js
# - Dockerfile
# - .dockerignore
touch package.json server.js Dockerfile .dockerignore
```

Add the following content to each file

`package.json`

```json
{
  "name": "docker_web_app",
  "version": "1.0.0",
  "description": "Node.js on Docker",
  "author": "First Last <first.last@example.com>",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

`server.js`

```js
'use strict';
 
const express = require('express');
 
// Constants
const PORT = 8080;
const HOST = '0.0.0.0';
 
// App
const app = express();
app.get('/', (req, res) => {
  res.send('Hello World');
});
 
app.listen(PORT, HOST, () => {
  console.log(`Running on http://${HOST}:${PORT}`);
});
```

`Dockerfile`

```Dockerfile
FROM node:18

# Create app directory
WORKDIR /usr/src/app

# Install app dependencies
COPY package*.json ./
RUN npm install

# Copy the rest of the source code
COPY . .

# The container will listen to the port 8080
EXPOSE 8080

# Command to startup this application
CMD [ "node", "server.js" ]
```

`.dockerignore`

```
node_modules
npm-debug.log
```

This is all the source code we need.
Now we can make use of the docker build command to build the image

```bash
# Build the image
docker build . -t nodeapp
```

Once the image is created, we can check the characteristics of it

```bash
# List images in local machine
docker images nodeapp
```

Let's now create a new container using the brand new image

```bash
# Run the new image
docker run -p 8080:8080 -d nodeapp
```

We can check if the container is up and running with by using the docker ps command

```bash
# In another tab/terminal check the status of the container
docker ps
```

We can execute the curl command to perform some http requests to the node application

```bash
# Test the application
curl localhost:8080
```

Let's stop and remove the container

```bash
# Stop the container
docker stop ID_CONTAINER

# Remove the container
docker rm ID_CONTAINER
```

## Container images part 2

In this example, we will create a new application that requires building a binary. We will use a simple golang code with a hello world message to show as output of the running application.

Create a new directory and inside create two new files, one for the Dockerfile and another for the golang code

```bash
mkdir -p ~/docker/build-image-golang
cd $_
touch Dockerfile main.go
```

Add the contents of the `Dockerfile` file:

```Dockerfile
FROM golang:1.21
WORKDIR /src
COPY main.go /src/main.go
RUN go build -o /bin/hello ./main.go

CMD ["/bin/hello"]
```

And also the contents of the `main.go` file:

```golang
package main

import "fmt"

func main() {
  fmt.Println("hello, world!")
}
```

Time to build the image with the `docker build` command

```bash
docker build . -t gohello:v1
```

We can check the information of the generated image

```bash
docker images gohello:v1
```

Once the image is built, we can run a container 

```bash
docker run gohello:v1
```

You can create `Dockerfile` files with multi-stage approach. This is appropiate for building applications that don't need the build tools to run. It allows you to generate lightweight images


```Dockerfile
# We name the first image build
FROM golang:1.21 as build
WORKDIR /src
COPY main.go /src/main.go
RUN go build -o /bin/hello ./main.go

# In the second stage we use scratch, which is an empty File System
FROM scratch
# Copy the resultant binary from the first stage
COPY --from=build /bin/hello /bin/hello
CMD ["/bin/hello"]
```

Build the image again

```bash
docker build . -t gohello:v2
```

We can check the information of the generated image

```bash
docker images gohello:v2
```

Once the image is built, we can run a container 

```bash
docker run gohello:v2
```

## Networking

By default, Docker comes with three network types:

- Default bridge
- host
- none

We will create a new network bridge and connect some containers to allow them have connectivity

We can list the current available networks for containers with the following command:

```bash
# List the networks
docker network list
```

Let's create a new bridge called mynet

```bash
# Create a new network
docker network create -d bridge mynet
```

If you list the available networks you can see the new one available

```bash
# List the networks again
docker network list
```

It is time to run a new container, but this time we will specify that it should be connected to that new bridge

```bash
# Create a new mysql container connected to the network
docker run --name some-mysql --net mynet -e MARIADB_ROOT_PASSWORD=mysecret -d mariadb:11
```

In a new tab or terminal, we will run a new container using the ubuntu image to run a mysql client from that host.

```bash
# Create a new ubuntu container to interact with the mysql server
# The flag --rm will remove the container automatically when it stops
docker run --rm -it --net mynet ubuntu bash
```

Once the container is running, we will:
- Update packages
- Install the mysql client
- Use the mysql client tool to connect remotely to the DB
- Create a new DB
- Install dnsutils
- Use the dig command to get the IP address of the host running the DB

```bash
# Update the packages
apt update

# Install the mysql client command
apt install -qqy default-mysql-client

# Connect to the mysql server from the ubuntu container
# Take into consideration that some-mysql is the name of the container
mysql -u root -p -h some-mysql
CREATE DATABASE workshops;
exit

# Install dnstools
apt install -qqy dnsutils

# Find the IP address of the mysql container
dig +short some-mysql
exit
```

Now that you know how to attach containers to a network, let's clean them up

```bash
# Remove the mysql container
docker rm --force some-mysql

# Remove the network
docker network rm mynet
```

**Exercise:**

- Run the same two containers, mariadb and ubuntu, without any custom network and you will see that the DNS does not work. You won't be able to retrieve the IP of the database from the ubuntu container.

## Manage volumes

Containers are ephimeral, it means they are meant to be recreated. But some applications are stateful, that is when you should mount directories.

In Docker, you can mount directories from your local Filesystem to the containers using two methods:

- Volumes. Docker manages the location of the files in the host.
- Bind mount. The user manages the location of the files in the host.

We will start first with volumes by creating a new one

```bash
# Create a new volume
docker volume create myvolume
# List the volumes
docker volume ls
```

We will run a new container that makes use of the new volume. It will be mounted in the directory `/root/myvolume` in the FS of the container

```bash
# Create a new ubuntu container that makes use of myvolume
docker run -it --name ubuntu-container -v myvolume:/root/myvolume/ ubuntu /bin/bash
```

Let's generate some files in the volume inside the container:

```bash
# Access to the directory in the container
cd /root/myvolume/

# Create a new file
touch newfile

# Add content to the file
echo writing something to the file > newfile 
cat newfile 

exit
```

After exiting the container we can remove it

```bash
docker rm ubuntu-container

# make sure that the container is not there anymore
docker ps -a
```

We can again create the same container and check if the files are still there

```bash
# recreate the container to remount the volume
docker run -it --name ubuntu-container -v myvolume:/root/myvolume/ ubuntu /bin/bash
```

If we chech the contents of the file we will see that the content is the same

```bash
# check if the file still exists
ls /root/myvolume/

# check the content of the file
cat /root/myvolume/newfile
exit
```

Now we can run an example more closer to a real scenario. We will create a new volume for a database

```bash
# Let's now test this with a mysql server
docker volume create dbvolume
```

We will run a container mount the new volume in `/var/lib/mysql` which is the location where the database is stored

```bash
# create the mysql server with the attached container
docker run --name some-mysql -v dbvolume:/var/lib/mysql -e MARIADB_ROOT_PASSWORD=mysecret -d mariadb:11
```

Once the container is up and running we can:
- Create a database
- Create a new table
- Insert a new record

This will help us to generate some data

```bash
# Inject a bash process to the previously created container
docker exec -it some-mysql bash

# Access to the database
mysql -u root -pmysecret

# Create and use a new database
CREATE DATABASE inventory;
USE inventory;

# Create a new table
CREATE TABLE Person (
  name VARCHAR(255),
  country VARCHAR(255),
  age INT
);

# Insert some records
INSERT INTO Person (name, country, age)
VALUES
    ('John', 'Netherlands', 42),
    ('Paul', 'Germany', 17),
    ('Mark', 'Italy', 21),
    ('Linda', 'United States', 34);

# exit from mysql
exit

# exit from the container
exit
```

Once we generated some data, it is time to remove the container!

```bash
# Remove the container
docker rm --force some-mysql
# check that the conatiner does not exist anymore
docker ps -a
```

We recreate the container with the very same command that we used previously

```bash
# Create again the same container using the same volume
docker run --name some-mysql -v dbvolume:/var/lib/mysql -e MARIADB_ROOT_PASSWORD=mysecret -d mariadb:11
```

Once the container is again up and running we can verify that our database is still accessible

```bash
# Inject again a bash process
docker exec -it some-mysql bash

# Access to the database
mysql -u root -pmysecret

# List the databases
SHOW DATABASES;

# Use the database inventory
USE inventory;

# List the tables and recods
SHOW TABLES;
SELECT * FROM Person;

exit
exit
```

Nice! Our data was still there. Now we can clean them up again.

```bash
docker rm --force some-mysql
docker volume rm dbvolume
```

## Manage bind mounts

With bind mounts, we can decide where is the code mounted in the FS of the host VM.

We will start first with a small exercise. We will mount an HTML file to an Apache container image. It will be served by the web server.

First create the new directory and the html file

```bash
mkdir ~/docker/bind-mount/
cd $_
touch index.html
```

This should be the content of `index.html`

```html
<!DOCTYPE html>
<html>
    <head>
        <title>Example</title>
    </head>
    <body>
        <p>This is an example of a simple HTML page with one paragraph.</p>
    </body>
</html>
```

Now run the container with the `-v` flag, as you can see, the first part of the argument, before the semicolon is the absolute location of the directory, in this case we make use of the variable `$PWD`, but we can also use the plain string

```bash
docker run -d --name apache -p 8000:80 -v $PWD:/usr/local/apache2/htdocs/ httpd:2.4
```

If we open the location http://localhost:8000 we will be able to see the html page that we mounted.

Let's remove the container

```bash
docker rm --force apache
```

Sometimes we don't need to build images to run our own containers, in some cases it is juste enough to mount files to have custom services.

Previously we created an image to run a node application. This time we will use bind mount to run a container using the node container image and mount the source code to run the desired application

```bash
# access to the code of the repo where the ptyhon app lives
cd ~/docker/build-image-node

# Run an interactive container
docker run -it --rm -v $PWD:/app/ -p 8080:8080 node:12 /bin/bash

# install the packages
cd /app
npm install

# run the application
node server.js

# in another terminal make an HTTP request with curl or open url localhsot:5050 in the browser
curl localhost:8080
```