# Docker-Laravel boilerplate

This is an example of dockerized laravel application with mysql and redis

## Getting Started

These will get you a copy of the project up and running on your local machine for development, testing and learning purposes

### Prerequisites

The only requirement to run this application is to have docker installed on your machine

### Installing

Enter in the docker folder and build the main application image

```
docker build -t laravel .
```

Run docker-compose

```
docker-compose up -d
```

Build the application

```
docker exec -it app composer install
```

Access the application in your browser

```
http://localhost
```

### Installing on Windows
Enter in the docker folder.
Enable path conversion from Windows-style to Unix-style in volume definitions. Users of Docker Machine and Docker Toolbox on Windows should always set this:
```
COMPOSE_CONVERT_WINDOWS_PATHS=0 
```

Build the main application image

```
docker build -t laravel .
```

Check app host port in docker-compose file (maybe host port 80 is not available):

```
  app:
    container_name: app
    image: laravel:latest
    environment:
      - VIRTUAL_HOST=app.local
    networks:
      - app
    ports: 
      - 80:80 <-- update. Ex: 7654:80
```	  

Run docker-compose

```
docker-compose up -d
```

Build the application

```
winpty docker exec -it app composer install
```

Post-installation steps:
- go to laravel application folder
- copy .env.example as .env file

Run following command:

```
winpty docker exec -it app php artisan key:generate
```

Access the application in your browser

```
http://localhost:7654
```

NOTE: If you expirience following error: 

```
 docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                       PORTS                               NAMES
52e710b29c54        laravel:latest      "script"                 6 seconds ago       Exited (127) 5 seconds ago                                       app

````

go to docker folder and edit script.sh, replacing \r\n with \n.

---------------------------------------------------------------------

## **Best practices for writing Dockerfile**

Docker builds images automatically by reading the instructions from a **Dockerfile** -- a text file that contains all commands, in order, needed to build a given image. A **Dockerfile** adheres to a specific format and set of instructions which you can find at [Dockerfile reference](https://docs.docker.com/engine/reference/builder/).

A Docker image consists of read-only layers each of which represents a Dockerfile instruction. The layers are stacked and each one is a delta of the changes from the previous layer. Consider this **Dockerfile:**

```dockerfile
FROM ubuntu:18.04
COPY . /app
RUN make /app
CMD python /app/app.py
```
Each instruction creates one layer:

- FROM creates a layer from the ubuntu:18.04 Docker image.
- COPY adds files from your Docker client’s current directory.
- RUN builds your application with make.
- CMD specifies what command to run within the container.

When you run an image and generate a container, you add a new writable layer (the “container layer”) on top of the underlying layers. All changes made to the running container, such as writing new files, modifying existing files, and deleting files, are written to this thin writable container layer.

## **General guidelines and recommendations**

### **Create ephemeral containers**

The image defined by your Dockerfile should generate containers that are as ephemeral as possible. By “ephemeral”, we mean that the container can be stopped and destroyed, then rebuilt and replaced with an absolute minimum set up and configuration.

### **Understand build context**

When you issue a **``docker build``** command, the current working directory is called the build context. By default, the Dockerfile is assumed to be located here, but you can specify a different location with the file flag (**``-f``**). Regardless of where the **Dockerfile** actually lives, all recursive contents of files and directories in the current directory are sent to the Docker daemon as the build context.

### Build context example

Create a directory for the build context and **``cd``** into it. Write “hello” into a text file named **``hello``** and create a Dockerfile that runs **``cat``** on it. Build the image from within the build context (**``.``**):

```bash
mkdir myproject && cd myproject
echo "hello" > hello
echo -e "FROM busybox\nCOPY /hello /\nRUN cat /hello" > Dockerfile
docker build -t helloapp:v1 .
```
Move **``Dockerfile``** and **``hello``** into separate directories and build a second version of the image (without relying on cache from the last build). Use **``-f``** to point to the Dockerfile and specify the directory of the build context:

```bash
mkdir -p dockerfiles context
mv Dockerfile dockerfiles && mv hello context
docker build --no-cache -t helloapp:v2 -f dockerfiles/Dockerfile context
```
Inadvertently including files that are not necessary for building an image results in a larger build context and larger image size. This can increase the time to build the image, time to pull and push it, and the container runtime size. To see how big your build context is, look for a message like this when building your **``Dockerfile:``**
```
Sending build context to Docker daemon  187.8MB
```

### **Exclude with .dockerignore**

To exclude files not relevant to the build (without restructuring your source repository) use a **``.dockerignore``** file. This file supports exclusion patterns similar to .**``gitignore``** files. For information on creating one, see the [.dockerignore file.](https://docs.docker.com/engine/reference/builder/#dockerignore-file)

### **Use multi-stage builds**

Multi-stage builds allow you to drastically reduce the size of your final image, without struggling to reduce the number of intermediate layers and files.

Because an image is built during the final stage of the build process, you can minimize image layers by leveraging build cache.

For example, if your build contains several layers, you can order them from the less frequently changed (to ensure the build cache is reusable) to the more frequently changed:

- Install tools you need to build your application

- Install or update library dependencies

- Generate your application

A Dockerfile for a Go application could look like:

```dockerfile
FROM golang:1.11-alpine AS build

# Install tools required for project
# Run `docker build --no-cache .` to update dependencies
RUN apk add --no-cache git
RUN go get github.com/golang/dep/cmd/dep

# List project dependencies with Gopkg.toml and Gopkg.lock
# These layers are only re-built when Gopkg files are updated
COPY Gopkg.lock Gopkg.toml /go/src/project/
WORKDIR /go/src/project/
# Install library dependencies
RUN dep ensure -vendor-only

# Copy the entire project and build it
# This layer is rebuilt when a file changes in the project directory
COPY . /go/src/project/
RUN go build -o /bin/project

# This results in a single layer image
FROM scratch
COPY --from=build /bin/project /bin/project
ENTRYPOINT ["/bin/project"]
CMD ["--help"]
```
### **Don’t install unnecessary packages**

To reduce complexity, dependencies, file sizes, and build times, avoid installing extra or unnecessary packages just because they might be “nice to have.” For example, you don’t need to include a text editor in a database image.

### **Minimize the number of layers**

In older versions of Docker, it was important that you minimized the number of layers in your images to ensure they were performant. The following features were added to reduce this limitation:

- Only the instructions **``RUN``**, **``COPY``**, **``ADD``** create layers. Other instructions create temporary intermediate images, and do not increase the size of the build.

- Where possible, use multi-stage builds, and only copy the artifacts you need into the final image. This allows you to include tools and debug information in your intermediate build stages without increasing the size of the final image.

### **Sort multi-line arguments**

Whenever possible, ease later changes by sorting multi-line arguments alphanumerically. This helps to avoid duplication of packages and make the list much easier to update. This also makes PRs a lot easier to read and review. Adding a space before a backslash (\) helps as well.

Here’s an example from the buildpack-deps image:

```dockerfile
RUN apt-get update && apt-get install -y \
  bzr \
  cvs \
  git \
  mercurial \
  subversion
  ```
### **Leverage build cache**

When building an image, Docker steps through the instructions in your **``Dockerfile``**, executing each in the order specified. As each instruction is examined, Docker looks for an existing image in its cache that it can reuse, rather than creating a new (duplicate) image.

If you do not want to use the cache at all, you can use the **```--no-cache=true```** option on the **```docker build```** command. However, if you do let Docker use its cache, it is important to understand when it can, and cannot, find a matching image. The basic rules that Docker follows are outlined below:

Starting with a parent image that is already in the cache, the next instruction is compared against all child images derived from that base image to see if one of them was built using the exact same instruction. If not, the cache is invalidated.

In most cases, simply comparing the instruction in the Dockerfile with one of the child images is sufficient. However, certain instructions require more examination and explanation.

For the **``ADD``** and ``**COPY**`` instructions, the contents of the file(s) in the image are examined and a checksum is calculated for each file. The last-modified and last-accessed times of the file(s) are not considered in these checksums. During the cache lookup, the checksum is compared against the checksum in the existing images. If anything has changed in the file(s), such as the contents and metadata, then the cache is invalidated.

Aside from the **``ADD``** and **``COPY``** commands, cache checking does not look at the files in the container to determine a cache match. For example, when processing a **``RUN apt-get -y update``** command the files updated in the container are not examined to determine if a cache hit exists. In that case just the command string itself is used to find a match.

Once the cache is invalidated, all subsequent **``Dockerfile``** commands generate new images and the cache is not used.

For more info about best practices for building docker images u can always visit the [official docker documentation](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)

### **Clean you images after installation layer**

Good practice if to clean you image after application and dependencies installation layers

clean up the apt cache by removing **``/var/lib/apt/lists``** it reduces the image size, since the apt cache is not stored in a layer. Since the **``RUN``** statement starts with **``apt-get update``**, the package cache is always refreshed prior to **``apt-get install``**.

Official Debian and Ubuntu images [automatically run](https://github.com/moby/moby/blob/03e2923e42446dbb830c654d0eec323a0b4ef02a/contrib/mkimage/debootstrap#L82-L105) **``apt-get clean``**, so explicit invocation is not required.