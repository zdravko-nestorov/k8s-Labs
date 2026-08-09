# 19 - Getting Started with Docker on Linux

> Course: Cloud Native Champions: CKAD Bootcamp  
> Module: Getting Started with Docker on Linux for AWS  
> Lab: AWS login / EC2 Instance Connect / Install Docker / docker group / CLI help / First container / First image  
> Topics: docker, docker-install, docker-group, docker-cli, docker-run, docker-ps, docker-exec, docker-logs, docker-search, dockerfile, docker-build, images, containers, layers, dockerhub, port-publishing, ec2

## Goal

Install Docker CE on Amazon Linux, run Docker without sudo, learn how the CLI is organised, run containers from Docker Hub images, then build your own image from a Dockerfile.

## Notes

### Docker editions

- **Community Edition (CE):** open source, free, the core engine. Previously called "Docker Engine". This lab uses CE.
- **Enterprise Edition (EE):** paid, adds support, a certification program for trusted containers, and more.

### Client and server

Docker uses a client-server design. The CLI talks to the **Docker daemon**, which does not have to run on the same host.

### Images and containers

- **Image:** a read-only snapshot built from layers. Each layer is read-only, so layers get a content hash and can be shared between images.
- **Container:** a running application on top of an image, with one extra **writable** layer. Many containers can share one image.
- `docker commit` turns a container's writable layer into an image layer.
- Usually one application or service per container.

### CLI shape

Two forms:

```bash
docker command-name [options]          # common command, for example: docker ps -a
docker management-group command [opts] # for example: docker container ls --all
```

Most common commands also exist under a management group. Some commands exist **only** under the group, for example `docker system df` and `docker system prune`.

Handy equivalences: `docker ps` = `docker container ls`, `docker ps -a` = `docker container ls --all`, `docker info` = `docker system info`.

### Registries, repositories, tags

- Default registry is **Docker Hub**.
- Images live in repositories named `account/repository`. Official images use the account `library`, for example `library/nginx`.
- **Tags** pick the version. With no tag, Docker uses `latest`.

### Useful run options

| Option | Meaning |
|--------|---------|
| `--name <name>` | Friendly container name instead of the id |
| `-d` | Detached, runs in the background and returns the shell |
| `-p host:container` | Publish a container port on a host port |
| `-it` | Interactive with a terminal, needed for shells |

### Dockerfile instructions used here

- `FROM` sets the base image layer
- `WORKDIR` sets the working directory inside the image
- `COPY . .` copies the build context into the image directory
- `RUN` runs a command in a new layer
- `EXPOSE` documents the listening port. It does **not** publish it.
- `CMD` sets the default command for containers made from the image

Each instruction can add a layer, and Docker creates a throwaway container per instruction, then commits the result. Keep the layer count reasonable; there is overhead and a hard limit.

The file must be named `Dockerfile`, capital D, rest lowercase.

### Security note on the docker group

Adding a user to the `docker` group is effectively giving that user root. Do it only for trusted users.

---

## Step 1 - Logging in to the AWS Console

1. Open the lab's cloud environment from the lab UI.

2. Sign in with the IAM user created for your lab session:

- Account ID or alias: keep the pre-populated value
- IAM user name: `student`
- Password: shown in the lab UI for that session

3. Select the **US West (Oregon) us-west-2** region in the upper-right dropdown. This lab requires us-west-2.

Use an up-to-date Chrome or Firefox.

---

## Step 2 - Connecting with EC2 Instance Connect

1. In the console search bar, type `EC2` and open the EC2 service.

2. Click **Instances** in the left menu. Look for the instance named `cloudacademylabs`.

If it is not there yet, the lab is still loading. Wait until Instance state is Running.

3. Right-click the instance and click **Connect**.

4. Select the **EC2 Instance Connect** tab. The Instance ID and public IP are shown.

5. In Username, enter `ec2-user` if the box is empty.

Note: no trailing space after `ec2-user`, or the connection fails.

6. Click **Connect**. A browser shell opens in a new window. Keep it open for the rest of the lab.

You can also use your own SSH client with the PPK or PEM key from the lab Credentials section.

---

## Step 3 - Installing Docker on Amazon Linux

1. Install Docker with yum:

```bash
sudo yum -y install docker
```

The transaction installs `docker` plus `containerd`, `libcgroup`, `pigz`, and `runc`, and updates `libseccomp`. In this run the versions were `docker 25.0.16-1.amzn2.0.3`, `containerd 2.1.9`, `runc 1.3.5`.

2. Start the service:

```bash
sudo systemctl start docker
```

3. Check it is running:

```bash
sudo docker info
```

Shortened output:

```text
Client:
 Version:    25.0.14
Server:
 Containers: 0
  Running: 0
  Paused: 0
  Stopped: 0
 Images: 0
 Server Version: 25.0.16
 Storage Driver: overlay2
 Cgroup Driver: cgroupfs
 Cgroup Version: 1
 Swarm: inactive
 Default Runtime: runc
 Kernel Version: 4.14.123-111.109.amzn2.x86_64
 Operating System: Amazon Linux 2
 Docker Root Dir: /var/lib/docker
```

Note the separate Client and Server versions. That is the client-server design.

### Summary (Step 3)

Docker CE is installed and running, and `docker info` prints system-wide information.

---

## Step 4 - Using Docker without root

1. Try the command without sudo:

```bash
docker info
```

It fails:

```text
ERROR: permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock
```

The daemon rejects users outside the `docker` group.

2. Check the group exists:

```bash
grep docker /etc/group
```

Expected: `docker:x:993:`. If the line is missing, create it:

```bash
sudo groupadd docker
```

3. Add your user:

```bash
sudo gpasswd -a $USER docker
```

The group file now shows `docker:x:993:ec2-user`, but `groups` still shows the old list, because group membership is cached for the session.

4. Start a shell with the new group active:

```bash
newgrp docker
```

`groups` now includes `docker`. Logging out and back in works too.

5. Confirm it works:

```bash
docker info
```

Note: if it still fails, restart the daemon with `sudo systemctl restart docker`.

### Summary (Step 4)

Members of the `docker` group can run Docker commands without sudo. Treat that as granting root.

---

## Step 5 - Getting help from the command line

1. List all commands:

```bash
docker --help
```

The output separates management commands from common commands.

2. Run a management command:

```bash
docker system info
```

Same output as `docker info`, because `info` is the common form of `docker system info`.

3. See what else is under `system`:

```bash
docker system --help
```

```text
Commands:
  df          Show docker disk usage
  events      Get real time events from the server
  info        Display system-wide information
  prune       Remove unused data
```

`df` and `prune` exist only under the group.

4. See the image commands:

```bash
docker image --help
```

```text
Commands:
  build       Build an image from a Dockerfile
  history     Show the history of an image
  import      Import the contents from a tarball to create a filesystem image
  inspect     Display detailed information on one or more images
  load        Load an image from a tar archive or STDIN
  ls          List images
  prune       Remove unused images
  pull        Download an image from a registry
  push        Upload an image to a registry
  rm          Remove one or more images
  save        Save one or more images to a tar archive
  tag         Create a tag TARGET_IMAGE that refers to SOURCE_IMAGE
```

5. See the container commands:

```bash
docker container --help
```

```text
Commands:
  attach   Attach local standard input, output, and error streams to a running container
  commit   Create a new image from a container's changes
  cp       Copy files/folders between a container and the local filesystem
  create   Create a new container
  diff     Inspect changes to files or directories on a container's filesystem
  exec     Execute a command in a running container
  export   Export a container's filesystem as a tar archive
  inspect  Display detailed information on one or more containers
  kill     Kill one or more running containers
  logs     Fetch the logs of a container
  ls       List containers
  pause    Pause all processes within one or more containers
  port     List port mappings or a specific mapping for the container
  prune    Remove all stopped containers
  rename   Rename a container
  restart  Restart one or more containers
  rm       Remove one or more containers
  run      Create and run a new container from an image
  start    Start one or more stopped containers
  stats    Display a live stream of container(s) resource usage statistics
  stop     Stop one or more running containers
  top      Display the running processes of a container
  unpause  Unpause all processes within one or more containers
  update   Update configuration of one or more containers
  wait     Block until one or more containers stop, then print their exit codes
```

### Summary (Step 5)

Docker groups commands by component. `--help` at any level shows what is available.

---

## Step 6 - Running your first container

1. Run the hello-world image:

```bash
docker run hello-world
```

Output has two parts. First Docker's own messages:

```text
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
4f55086f7dd0: Pull complete
Digest: sha256:7f4da0fc94bcece205a8c0b6f4d11c8196924654ffe5c4d1aa439b7f632048b2
Status: Downloaded newer image for hello-world:latest
```

Then the container's own output, starting `Hello from Docker!`.

Reading the Docker part: the image was not local, so Docker pulled `library/hello-world` with the default `latest` tag. You could pull it yourself with `docker pull hello-world`. You gave no command, so the image's default command ran.

2. Run it again:

```bash
docker run hello-world
```

No pull messages this time. The image is already local.

3. Run nginx with options:

```bash
docker run --name web-server -d -p 8080:80 nginx:1.12
```

Three `Pull complete` lines means the image has three layers. The last line is the container id.

Options used: `--name web-server` names it, `-d` detaches, `-p 8080:80` maps host 8080 to container 80.

4. Test it:

```bash
curl localhost:8080
```

You get the nginx welcome HTML.

5. List running containers:

```bash
docker ps
```

```text
CONTAINER ID   IMAGE        COMMAND                  CREATED         STATUS         PORTS                                   NAMES
0249c1c76dc1   nginx:1.12   "nginx -g 'daemon of…"   2 minutes ago   Up 2 minutes   0.0.0.0:8080->80/tcp, :::8080->80/tcp   web-server
```

6. List all containers, including stopped ones:

```bash
docker ps -a
```

```text
CONTAINER ID   IMAGE         COMMAND                  CREATED         STATUS                     PORTS                                   NAMES
0249c1c76dc1   nginx:1.12    "nginx -g 'daemon of…"   3 minutes ago   Up 3 minutes               0.0.0.0:8080->80/tcp, :::8080->80/tcp   web-server
76b830871dfd   hello-world   "/hello"                 3 minutes ago   Exited (0) 3 minutes ago                                           blissful_albattani
24e702522c33   hello-world   "/hello"                 7 minutes ago   Exited (0) 7 minutes ago                                           nervous_euclid
```

The hello-world containers exited after printing. nginx keeps listening. Docker invents random names when you do not pass `--name`.

7. Stop nginx:

```bash
docker stop web-server
```

8. Confirm it is gone from `docker ps`. `curl localhost:8080` now fails too.

9. Start the same container again:

```bash
docker start web-server
```

This reuses the stopped container. `docker run` would create a second one.

10. Read its logs:

```bash
docker logs web-server
```

```text
172.17.0.1 - - [09/Aug/2026:13:45:20 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/8.3.0" "-"
```

Docker collects standard output and standard error.

11. Open a shell inside the container:

```bash
docker exec -it web-server /bin/bash
```

The prompt becomes `root@0249c1c76dc1:/#`. Try `ls` and `cat /etc/nginx/nginx.conf`, then `exit`.

Bash exists here because the nginx image has a Debian layer. Not every image has bash.

12. Run a single command without a shell:

```bash
docker exec web-server ls /etc/nginx
```

13. Stop the container:

```bash
docker stop web-server
```

14. Search Docker Hub from the CLI:

```bash
docker search "Microsoft .NET Core"
```

It returns names, descriptions, stars, and whether the image is official. Good for recalling a name, weak for images with many configuration options.

15. For those, use the Docker Hub website, for example the .NET repository page. It lists supported tags and a Dockerfile per tag group, plus usage documentation.

### Summary (Step 6)

You ran containers, published a port, listed, stopped, started, read logs, executed commands inside a container, and searched for images.

---

## Step 7 - Creating your first Docker image

Two ways to build an image: `docker commit` from a changed container, or a **Dockerfile**. A Dockerfile is easier to maintain, repeat, and share. This step uses a Python Flask app.

1. Install Git:

```bash
sudo yum -y install git
```

2. Clone the app:

```bash
git clone https://github.com/cloudacademy/flask-content-advisor.git
```

3. Enter the directory:

```bash
cd flask-content-advisor
```

4. Create the Dockerfile:

```bash
nano Dockerfile
```

Note: the name must be `Dockerfile`, uppercase D and the rest lowercase.

5. Enter this content:

```dockerfile
# Python v3 base layer
FROM python:3
# Set the working directory in the image's file system
WORKDIR /usr/src/app
# Copy everything in the host working directory to the container's directory
COPY . .
# Install code dependencies in requirements.txt
RUN pip install --no-cache-dir -r requirements.txt
# Indicate that the server will be listening on port 5000
EXPOSE 5000
# Set the default command to run the app
CMD [ "python", "src/app.py" ]
```

Full instruction list: the Dockerfile reference page.

6. Save with Ctrl+X, then `Y`, then Enter.

7. Build the image:

```bash
docker build -t flask-content-advisor:latest .
```

`-t` names and tags the image. The `.` sets the build context, so Docker looks for a Dockerfile in the current directory.

Shortened output:

```text
 => [internal] load build definition from Dockerfile
 => [internal] load metadata for docker.io/library/python:3
 => [1/4] FROM docker.io/library/python:3@sha256:3a9d2dd3...
 => [2/4] WORKDIR /usr/src/app
 => [3/4] COPY . .
 => [4/4] RUN pip install --no-cache-dir -r requirements.txt
 => exporting to image
 => => writing image sha256:e8397fe465a3...
 => => naming to docker.io/library/flask-content-advisor:latest
```

Step 1 is slow because it pulls the Python base layers. Step 4 is slow because it downloads dependencies.

8. Get the VM public IP:

```bash
curl ipecho.net/plain; echo
```

9. Open that IP in a new browser tab. Nothing loads yet, since no server runs. Keep the tab open.

10. Run a container from your image:

```bash
docker run --name advisor -p 80:5000 flask-content-advisor
```

Host port 80 maps to container port 5000. No `-d`, so the shell stays attached and you see the Flask output:

```text
 * Serving Flask app 'app'
 * Debug mode: on
WARNING: This is a development server. Do not use it in a production deployment.
 * Running on all addresses (0.0.0.0)
 * Running on http://127.0.0.1:5000
 * Running on http://172.17.0.2:5000
```

With `-d` you would read the same through `docker logs`.

11. Refresh the browser tab. The advisor page appears.

12. Back in the shell, the requests are logged:

```text
<your-public-ip> - - [09/Aug/2026 14:03:37] "GET / HTTP/1.1" 200 -
<your-public-ip> - - [09/Aug/2026 14:03:37] "GET /favicon.ico HTTP/1.1" 404 -
```

Two requests per page load, because the browser also asks for a favicon.

### Validation (Step 7)

- **EC2 Instance Serves a Website Through Docker:** the instance public IP returns HTTP 200

### Summary (Step 7)

You wrote a Dockerfile, built an image with `docker build`, and ran a container from it.

---

## Summary (all steps)

Install Docker CE, add your user to the `docker` group, then use the CLI: `docker run` to start containers, `-p` to publish ports, `ps`, `logs`, `exec`, `stop`, and `start` to work with them. Images come from Docker Hub repositories with tags, or you build your own from a Dockerfile with `docker build -t name:tag .`.
