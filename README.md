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
