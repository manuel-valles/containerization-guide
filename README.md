# Containerization Guide

Build, test and deploy container with _Docker_, _Kubernetes_, _Compose_, _Swarm_ and _Registry_ using **DevOps**.

## Docker

### Basic Docker Commands

- `$ docker version` : verifies CLI can talk to engine
- `$ docker info` : most config values of engine
- `$ docker <command> <sub-command>` : CLI structure
  - `$ docker container run --publish 80:80 --detach --name webhost nginx` : will start and run a new container from the _Nginx_ image in the background (`--detach` or `-d`). It will open a port (`--publish` or `-p`) 80 on the [host IP](http://localhost/). You can use any port available on the left, e.g. `8088:80`
  - `$ docker container run --publish 3306:3306 -d --name db -e MYSQL_RANDOM_ROOT_PASSWORD=yes mysql`: will start and run a new container with the environment variables provided (`--env` or `-e`)
  - `$ docker container ls` : list our running containers. If you add the flag `-a` it will return all the existing containers. You can also use `$ docker container ps`, and locally/host just `$ ps aux` or `$ ps aux | grep webhost` for a specific one
  - `$ docker container stop 12656f9f7de3` : stops a container. Just providing the first 3 digits is enough
  - `$ docker container logs webhost` : logs for the specific docker container
  - `$ docker container top webhost` : processes are running in a container
  - `$ docker container inspect webhost` : details of one container config
  - `$ docker container rm -f f14 126 6bb`: removes the provided containers IDs. The flag `-f` will stop and remove any running container
  - `$ docker container stats` : performance stats for all containers
  - `$ docker container run -it --name proxy nginx bash` : `-it` is a combination of two flags: `-i` keeps session open to receive terminal input; and `-t` allocates a pseudo `-TTY` (**No SSH needed** since the Docker CLI is great substitute for adding SSH to containers). This alongside with the command `bash` will give you a terminal inside the running container. You could use any other like Ubuntu: `$ docker container run -it --name ubuntu ubuntu`
  - `$ docker container start -ai ubuntu` : runs the previous container with the existing bash, instead of creating a new one
  - `$ docker container exec -it db bash` : runs additional process in the running container. The `ps` is no included in the `mysql` image by default, so you can do so inside the container with `# apt-get update && apt-get install -y procps`

### Basic Container Concepts

- An image is the application we run
- A container is an instance of that image running as a process
- You can have many containers running off the same image
- Containers aren't mini-VMs, they are just processes limited to what resources they can access
- _Docker's_ default image **registry** is called [Docker Hub](https://hub.docker.com/)

### Docker Networks

- Each container connected to a private virtual network `bridge`
- Each virtual network routes through NAT firewall on host IP
- All containers on a virtual network can talk to each other without `-p`
- Docker networks defaults work well in many cases but easy to swap out parts to customize it
- Use different Docker network drivers to gain new abilities
- `$ docker container port webhost`
- `$ docker container inspect --format '{{ .NetworkSettings.IPAddress }}' webhost` : `--format` is a common option for formatting the output of commands using `Go templates`. If you run `$ ifconfig en0`, you will be able to see that the hosts are not matching
- Main management commands:
  - `$ docker network ls` : shows networks
  - `$ docker network inspect` : inspects a network
  - `$ docker network create my_app_net` : spawns a new virtual network for you to attach containers to
  - `$ docker network connect my_app_net webhost` : attaches a network to container. We can check if that works with: `$ docker container inspect webhost`
  - `$ docker network disconnect my_app_net webhost` : detaches a network from container
- Default security:
  - Create your apps so frontend/backend sit on same Docker network
  - Their inter-communication never leaves host
  - All externally exposed ports closed by default, so you must manually expose via `-p` which is better default security!
- DNS:
  - DNS is the key to easy inter-container communications, and Docker daemon has a built-in DNS server that containers use by default. Containers shouldn't rely on IP's for inter-communication.
  - `$ docker container run -d --name my_nginx --network my_app_net nginx:alpine`
  - `$ docker container exec -it my_nginx ping webhost` : will show if there is actual communication between containers
  - Example:
    - `$ docker network create robin_test` : creates a virtual network with the default bridge driver
    - `$ docker container run -d --net robin_test --net-alias search elasticsearch:2` : you should run this twice. `--net-alias`/`--network-alias` provides an additional DNS name to respond to
    - `$ docker container run --rm --net robin_test alpine nslookup search` : to see the two containers list for the same DNS name
    - `$ docker container run --rm --net robin_test centos curl -s search:9200` : will retrieve the two DNS randomly

_NOTE_: The official `nginx` image (`nginx` or `nginx:latest` ) has removed `ping`. So to be able to use this command you can download the `alpine` version.

### Container Images

- An image is an ordered collection of root filesystem changes and the corresponding execution parameters for use within a container runtime. In other words, it is the binaries and dependencies for your app and the metadata on how to run it
- `$ docker image ls` : shows images
- `$ docker pull nginx` : pulls the latest image of nginx
- `$ docker pull nginx:1.21.6` : pulls a specific version image of nginx
- `$ docker pull nginx:1.21.6-alpine` : pulls the alpine image of nginx - `alpine` is the smallest sized linux distribution
- `$ docker history nginx:latest` :retrieves the different layers/changes/differences of the image. Each layer is uniquely identified and only stored once on a host what saves storage space and transfer time on push/pull
- `$ docker image inspect nginx` : retrieves the metadata of the image
- A container is just a single read/write layer on top of an image
- `$ docker image tag --help` : info about image tags. The default tag is the _latest_ if not specified. Otherwise: `<user>/<repo>:<tag>`. The official repositories live at the "root namespace" of the registry, so they don't need account name in front of the repo name
- `$ docker image tag nginx manukem/nginx` : creates a tag. This will create the default `latest`, unless you specify one: `docker image tag manukem/nginx manukem/nginx:testing`
- `$ docker image push manukem/nginx` : pushes the image to your personal repositories. Note that you will have to log in to be able to push it - `$ docker login`. Remember to logout from shared machines or servers when done to protect your account: `$ docker logout`
- **Dockerfile** is a recipe to create docker images which filename is standardised but you could use any other name with the flag `-f`: `$ docker build -f some-dockerfile`
  - Package manager like _apt_ and _yum_ are one of the reasons to build containers from Debian, Ubuntu, Fedora or CentOS, e.g. `FROM debian:stretch-slim`
  - One of the reason **environment variables** were chosen as preferred way to inject key/value is they work everywhere, on every OS and config, e.g. `ENV NGINX_VERSION 1.13.6-1~stretch`
  - The order of the layer matters, so all the `RUN` commands must be in order from top to down and separated by double ampersands (`&&`)
  - Docker handles all the logs for us, so we just need to link the `stdout` and `stderr` to the `access.log` and `error.log`
  - To expose some ports on the Docker virtual network you should add a `EXPOSE` line. However, you still need to use `-p` or `-P` to open/forward them on host
  - A last `CMD` is required and it will run when the container is launched. NOTE: Only one _CMD_ is allowed, so if there are multiple, last one wins!
  - `$ docker image build -t customnginx .` : Run the _Dockerfile_ and provide a repository tag
  - If you can use an official image, it would be recommended because it would be easier to maintain
  - `WORKDIR` makes easier to describe where you are going and it is also preferred to using `RUN cd /some.path`
  - `COPY` is the way to copy your source code from your local machine into the container images
- You can use **prune** commands to clean up images, volumes, build cache, and containers. For example:
  - `$ docker image prune` cleans up just "dangling" images
  - `$ docker system prune` cleans up everything
  - `$ docker image prune -a` removes all images you are not using
  - `$ docker system df` to see space usage
