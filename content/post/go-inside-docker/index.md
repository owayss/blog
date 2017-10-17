---
title: "Go in Dockerland"
description: "Using Docker for building and deploying Go servers"
tags: [
    "containers",
	"go",
]
date: 2017-09-12
---
There are a couple of things that need to be done before one can get a Go development environment ready on a host machine: the actual Go compiler needs to be installed, the `$GOPATH` and `$GOROOT` environment variables need to be set, and the workspace directory must be set up so that Go projects can follow an expected directory structure.

The setup process can be tedious for some users, and certainly requires some time.

## Enter Docker
One of the -many- reasons why you might want to use Docker is to make your development and build environment faster, more efficient, and more lightweight. The only thing that a developer need to have installed on their host machine is Docker itself; everything else will be running inside a Docker container.


For the purposes of this post, a simple Hello World web application would do.
```go
package main

import (
	"io"
	"log"
	"net/http"
)

func hello(w http.ResponseWriter, r *http.Request) {
	io.WriteString(w, "Hello, world!")
}

func main() {
	log.Print("Listening on http://localhost:4040")
	http.HandleFunc("/", hello)
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```
## The straightforward approach
Docker containers are launched from images. The standard way of creating an image is by writing a Dockerfile, which can be thought of as the "source code" for a container.

As our first approach, let us use the one from the [Go blog](https://blog.golang.org/docker):

```docker
# Start from a Debian image with the latest version of Go installed
# and a workspace (GOPATH) configured at /go.
FROM golang

# Copy the local package files to the container's workspace.
ADD . /go/src/github.com/user/myapp

# Build the app command inside the container.
# (You may fetch or manage dependencies here,
# either manually or with a tool like "godep".)
RUN go install github.com/user/myapp

# Run the app command by default when the container starts.
ENTRYPOINT /go/bin/myapp

# Document that the service listens on port 8080.
EXPOSE 8080
```

To build the image, just execute the following command:

`docker build -t myapp .`
```shell
Sending build context to Docker daemon  3.584kB
Step 1/5 : FROM golang
 ---> 1cdc81f11b10
Step 2/5 : ADD . /go/src/github.com/user/myapp
 ---> 187ab3c3b89e
Step 3/5 : RUN go install github.com/user/myapp
 ---> Running in 5f4daa2fc869
 ---> 69ebde31e61c
Removing intermediate container 5f4daa2fc869
Step 4/5 : ENTRYPOINT /go/bin/myapp
 ---> Running in 95b846f62fd0
 ---> ce2c15ad6b2b
Removing intermediate container 95b846f62fd0
Step 5/5 : EXPOSE 8080
 ---> Running in 5d2da1c6a0f5
 ---> 8b48be229a7b
Removing intermediate container 5d2da1c6a0f5
Successfully built 8b48be229a7b
Successfully tagged myapp:latest
```
To launch a container from that image, we just run:

`docker run --publish 8080:8080 --name myapp_test --rm myapp`

This will run a docker container from the `myapp` image we built earlier, forward the container's port 8080 to the host machine's 8080 port, and set the container's name to `myapp_test`.

Our server should be listening at port 8080:
```sh
curl http://localhost:6060
Hello, world!%
```

## How are Docker images built
Let us take a look into the image we created earlier:
```sh
docker images myapp --format "table {{.ID}}\t{{.Repository}}\t{{.Size}}"
IMAGE ID            REPOSITORY          SIZE
8b48be229a7b        myapp               735MB
```

As you can see in the output of the `docker images` command, the size of the image is 735MB. That is an awful lot for a hello world app!

Essentially, a Docker image is made up of filesystems layered over each other. At the base is a boot filesystem, `bootfs`, which resembles the typical Linux/Unix boot filesystem. On top of `bootfs`, Docker layers a `rootfs`, which, in the case of our `golang:latest` image, is a Debian filesystem.


![Docker image layers](container-layers.jpg "Docker image layers")


But, do we _really_ need a complete Linux image to run our Go server?


## A better way to do it

One of the nicest features of the Go language is that Go binaries can be statically linked. So the key idea here will be to separate the build process from the production environment; to build the Go application in one container, possibly using the same image we created earlier, and then ship the resulting statically linked binary in a different container; a much smaller one.

### The build container
We will use the container we built earlier to build our Go app with static linking enabled:
```sh
docker exec myapp_test CGO_ENABLED=0 GOOS=linux \
	go build -a -tags myapp -ldflags "-w -s" -o .
```
The `docker exec` command lets us run a process inside an already running container. Here, the process is the `go build` command to build our Go project and we will be running it inside our `myapp_test` container.

`-ldflags`: the `-s` tells the go tool that this needs to be built as a static binary, the `-w` turns off the DWARF debugging information.  

### The production container
Now we can just copy the binary we built in the previous step to our host using `docker cp`:
```sh
docker cp `docker ps -aqf "name=myapp_test"`:/github.com/user/myapp/myapp .
```

Since we have statically linked executable, we can just use the `scratch` image.

`scratch` is a reserved minimal Docker image, it is literally of size 0, and it is used to let the build process know that we want the next command in the Dockerfile to be the first filesystem layer in our container image.

Here is what the Dockerfile.prod for our minimal container looks like:
```dockerfile
FROM scratch
ADD myapp myapp
EXPOSE 8080
ENTRYPOINT ["/myapp"]
```

To build an image from that file:
```sh
docker build -t myapp_prod -f Dockerfile.prod
```

To launch a container from that image:
```sh
docker run --publish 8080:8080 --name prod --rm myapp_prod
```