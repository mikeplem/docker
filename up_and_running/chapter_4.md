# Chapter 4 - Working with Docker Images

## Key Items

- Each line in a Dockerfile creates a new image layer
    - This is beneficial because it means that Docker only needs to build layers that have changed
- While it is possible for you to build a container from a plain base image, it may be a good idea to look up what images already exist on public container registires
    - [Docker Hub](https://hub.docker.com/)
        - The defacto standard
        - For our purposes, it may not be what we want
        - Docker hub, rightly so, has rate limits that can hinder deployments
    - [Amazon Elastic Container Registry](https://gallery.ecr.aws/)
        - May never have as many images that would be found on Docker Hub
        - Not affected by rate limits that we have encountered
- The order of commands in your Dockerfile can impact the speed of builds. Move the commands that change to the bottom of the Dockerfile. This will limit the number of steps that have to be performed on subsequent builds
- It is best to only run a single process inside a container rather that more than one. This makes the container much easier to scale horizontally.
    - For example, do not run a database and frontend application inside the same container
- If you want to ensure all future Docker builds never use cache, use the `--no-cache` argument on the build command line.
    - This is a good troubleshooting steps to take if the final artifact is not working exactly as you expect when all other code has not changed.

## Dockerfile Walkthrough

Please review the [canonical reference guide](https://docs.docker.com/reference/dockerfile/) for all Dockerfile commands.

### FROM

- The base image on which all commands will be executed

### ARG

- Only used during build time
- Provides a way to set variables and default values

### LABEL

- Adds metadata to a container which can be used to search for and identify containers

### USER

- The user inside the container the next set of commands will run as
- By default, the `root` user runs all the commands
- Where possible, it is best to run the resulting applicaiton as a non root user

### ENV

- These are environment variables
- Allows you to set shell variables which can be used by the running application
- Using environment variables can make the Dockerfile simpler
    - By setting an ENV var you can make your file simpler to read by not having to hard code a path or value multiple times

Before setting an ENV variable

```
FROM node:20

RUN mkdir -p /app/config/
COPY ./config/dev.json /app/config/config.json
RUN chmod 644 /app/config/config.json
```

After setting an ENV variable

```
FROM node:20

ENV CFG_DIR="/app/config"

RUN mkdir -p $CFG_DIR
COPY ./config/dev.json $CFG_DIR/config.json
RUN chmod 644 $CFG_DIR/config.json
```

### RUN

- Used to execute programs inside the container
- This may be one of the most used commands
- Since each line in a Dockerfile is a new image layer, it may be a good idea to spend a little time thinking about the commands executed on a RUN line
    - Try to keep commands that relate to each other on a RUN line
    - This way you take advantage of future caching

This is a contrived example but gives and idea of what a command RUN command looks like. This first example will create 3 different layers for software updates and installation. If an image were to be built multiple times a day there is a very high chance none of the values will change that often.

```
FROM node:20

RUN apt-get update -y
RUN apt-get upgrade -y
RUN apt-get install -y curl

COPY ./path/from/source /path/to/target
```

By combining the three RUN lines into one the you ensure there will only be one cached imaged layer.

```
FROM node:20

RUN apt-get update -y && apt-get upgrade -y && apt-get install -y curl

COPY ./path/from/source /path/to/target
```

### COPY

- How you get files from the build computer into the container

### WORKDIR

- Changes the working directory for all subsequent commands
- By default, the Dockerfile runs commands in the `/` directory inside the container.

### CMD

- This defines the command run when a container starts

## Building an Image

**NOTE:** The examples below are being run using Podman on Linux which is a daemonless drop in replacement for Docker. However, Podman does not work as well on macOS. Some of the output may look a little different than what you see.

- Clone the example git repo

```
git clone https://github.com/spkane/docker-node-hello.git --config core.autocrlf=input
```

- Run your first build

```
$ docker image build -t example/docker-node-hello:latest .
STEP 1/15: FROM docker.io/node:18.13.0
Trying to pull docker.io/library/node:18.13.0...
Getting image source signatures
Copying blob 8bc43c905b24 done  
Copying blob 9bd150679dbd done  
Copying blob 56261d0e6b05 done  
Copying blob f049f75f014e done  
Copying blob bbeef03cda1f done  
Copying blob 5b282ee9da04 done  
Copying blob 5201db2cd6c6 done  
Copying blob 2a7091f85153 done  
Copying blob 2bcce6ea6105 done  
Copying config b68a472583 done  
Writing manifest to image destination
Storing signatures
STEP 2/15: ARG email="anna@example.com"
--> 9d2e5b57201
STEP 3/15: LABEL "maintainer"=$email
--> 8cb1d8384e2
STEP 4/15: LABEL "rating"="Five Stars" "class"="First Class"
--> 5440f164b30
STEP 5/15: USER root
--> c00803c5ecb
STEP 6/15: ENV AP /data/app
--> 4e85b4bb6dd
STEP 7/15: ENV SCPATH /etc/supervisor/conf.d
--> 53915506464
STEP 8/15: RUN apt-get -y update
Get:1 http://deb.debian.org/debian bullseye InRelease [116 kB]
Get:2 http://deb.debian.org/debian-security bullseye-security InRelease [27.2 kB]
Get:3 http://deb.debian.org/debian bullseye-updates InRelease [44.1 kB]
Get:4 http://deb.debian.org/debian bullseye/main amd64 Packages [8066 kB]
Get:5 http://deb.debian.org/debian-security bullseye-security/main amd64 Packages [317 kB]
Get:6 http://deb.debian.org/debian bullseye-updates/main amd64 Packages [18.8 kB]
Fetched 8589 kB in 1s (6320 kB/s)
Reading package lists...
--> 81c46e2616c
STEP 9/15: RUN apt-get -y install supervisor
Reading package lists...
Building dependency tree...
Reading state information...
The following additional packages will be installed:
  python3-pkg-resources
Suggested packages:
  python3-setuptools supervisor-doc
The following NEW packages will be installed:
  python3-pkg-resources supervisor
0 upgraded, 2 newly installed, 0 to remove and 132 not upgraded.
Need to get 499 kB of archives.
After this operation, 2320 kB of additional disk space will be used.
Get:1 http://deb.debian.org/debian-security bullseye-security/main amd64 python3-pkg-resources all 52.0.0-4+deb11u1 [190 kB]
Get:2 http://deb.debian.org/debian bullseye/main amd64 supervisor all 4.2.2-2 [309 kB]
debconf: delaying package configuration, since apt-utils is not installed
Fetched 499 kB in 1s (945 kB/s)
Selecting previously unselected package python3-pkg-resources.
(Reading database ... 22790 files and directories currently installed.)
Preparing to unpack .../python3-pkg-resources_52.0.0-4+deb11u1_all.deb ...
Unpacking python3-pkg-resources (52.0.0-4+deb11u1) ...
Selecting previously unselected package supervisor.
Preparing to unpack .../supervisor_4.2.2-2_all.deb ...
Unpacking supervisor (4.2.2-2) ...
Setting up python3-pkg-resources (52.0.0-4+deb11u1) ...
Setting up supervisor (4.2.2-2) ...
invoke-rc.d: could not determine current runlevel
invoke-rc.d: policy-rc.d denied execution of start.
--> e8423d5575d
STEP 10/15: RUN mkdir -p /var/log/supervisor
--> 73cf52561e7
STEP 11/15: COPY ./supervisord/conf.d/* $SCPATH/
--> b2fe62f6978
STEP 12/15: COPY *.js* $AP/
--> 63a3d020ee3
STEP 13/15: WORKDIR $AP
--> f12a5ac9f63
STEP 14/15: RUN npm install
npm WARN EBADENGINE Unsupported engine {
npm WARN EBADENGINE   package: 'formidable@1.0.13',
npm WARN EBADENGINE   required: { node: '<0.9.0' },
npm WARN EBADENGINE   current: { node: 'v18.13.0', npm: '8.19.3' }
npm WARN EBADENGINE }
npm WARN deprecated mkdirp@0.3.4: Legacy versions of mkdirp are no longer supported. Please update to mkdirp 1.x. (Note that the API surface has changed to use Promises in 1.x.)
npm WARN deprecated formidable@1.0.13: Please upgrade to latest, formidable@v2 or formidable@v3! Check these notes: https://bit.ly/2ZEqIau
npm WARN deprecated connect@2.7.9: connect 2.x series is deprecated

added 18 packages, and audited 19 packages in 969ms

8 vulnerabilities (1 low, 1 moderate, 6 high)

To address all issues (including breaking changes), run:
  npm audit fix --force

Run `npm audit` for details.
npm notice 
npm notice New major version of npm available! 8.19.3 -> 10.9.1
npm notice Changelog: <https://github.com/npm/cli/releases/tag/v10.9.1>
npm notice Run `npm install -g npm@10.9.1` to update!
npm notice 
--> 2f8d4ac4024
STEP 15/15: CMD ["supervisord", "-n"]
COMMIT example/docker-node-hello:latest
--> f25dcfc6637
Successfully tagged localhost/example/docker-node-hello:latest
f25dcfc6637690e93eb2711c54e8441ddd19aae4f575e057a0bae51f0805aca1
```

This is what the output looks like if you run the command a second time. Everything is already cached so the process runs much faster.

```
$ docker image build -t example/docker-node-hello:latest .
STEP 1/15: FROM docker.io/node:18.13.0
STEP 2/15: ARG email="anna@example.com"
--> Using cache 9d2e5b57201a4ca7414b65f7a0b5162f0a38e267d6918efdc87cf6a76b76bad7
--> 9d2e5b57201
STEP 3/15: LABEL "maintainer"=$email
--> Using cache 8cb1d8384e218bdd24f07d84cd3d0dc06c5ad6c10b25a6e3fa26da31842d27b5
--> 8cb1d8384e2
STEP 4/15: LABEL "rating"="Five Stars" "class"="First Class"
--> Using cache 5440f164b3094b4b4e13bccfd6d951d5a72c26abab0c9304b95dda45b55b2248
--> 5440f164b30
STEP 5/15: USER root
--> Using cache c00803c5ecb034f64a2d417749dfca17e9890806d9e9f921d3283bbb2afec977
--> c00803c5ecb
STEP 6/15: ENV AP /data/app
--> Using cache 4e85b4bb6dd90994d03727895c16b0c8844f0aefa3b8733c2836e21b4f9b85eb
--> 4e85b4bb6dd
STEP 7/15: ENV SCPATH /etc/supervisor/conf.d
--> Using cache 53915506464ece8515857da9bb0c8303f25dfe4f66acdeb9119e88d0f16aa909
--> 53915506464
STEP 8/15: RUN apt-get -y update
--> Using cache 81c46e2616c16ff8f4b2f181475dc577ba32fb63a357a43dbc0a55a18a8230cf
--> 81c46e2616c
STEP 9/15: RUN apt-get -y install supervisor
--> Using cache e8423d5575dc6ec834bf61b0b099696b77c1fc4df090ee07e0d0be13b6415c04
--> e8423d5575d
STEP 10/15: RUN mkdir -p /var/log/supervisor
--> Using cache 73cf52561e7d2925430695ee92b5461e2c0f4d9c50a19b859ad0ea3029cfdc94
--> 73cf52561e7
STEP 11/15: COPY ./supervisord/conf.d/* $SCPATH/
--> Using cache b2fe62f697823430f1c6a3aa2760eeff07b81d9ed6a27c7d24dd61453fe2b677
--> b2fe62f6978
STEP 12/15: COPY *.js* $AP/
--> Using cache 63a3d020ee3f4ba943c5b71777b481749cefe810ebd3d6d23ed3f57290b20349
--> 63a3d020ee3
STEP 13/15: WORKDIR $AP
--> Using cache f12a5ac9f631618fc499dca2f9d037c54f238f9d22b2f8c37acbd73dd7b6cd67
--> f12a5ac9f63
STEP 14/15: RUN npm install
--> Using cache 2f8d4ac4024d7a10a04200d1e3739bc709618eacad5eda8d55b261af5540fb4a
--> 2f8d4ac4024
STEP 15/15: CMD ["supervisord", "-n"]
--> Using cache f25dcfc6637690e93eb2711c54e8441ddd19aae4f575e057a0bae51f0805aca1
COMMIT example/docker-node-hello:latest
--> f25dcfc6637
Successfully tagged localhost/example/docker-node-hello:latest
f25dcfc6637690e93eb2711c54e8441ddd19aae4f575e057a0bae51f0805aca1
```

## Running Your Image

```
$ docker container run --rm -d -p 8080:8080 example/docker-node-hello:latest
696595178c682e7237eee231bf2da0df86a644563e0a9f77a17de0e8ff1c44ac
```

- `--rm` will make sure the image is removed when the container is stopped
- `-d` runs the image in the background or `detached` mode
- `-p` is the local and container port the application is listening. The left side of the colon is the port your local computer is listening and the right side of the colon is the port your applicaiton is running. These port numbers do not have to match

```
$ docker container ls
CONTAINER ID  IMAGE                                       COMMAND         CREATED        STATUS            PORTS                   NAMES
696595178c68  localhost/example/docker-node-hello:latest  supervisord -n  2 minutes ago  Up 2 minutes ago  0.0.0.0:8080->8080/tcp  cool_knuth
```

We can use the `curl` command to access the container

```
curl http://localhost:8080
Hello World. Wish you were here.
```

This is not in the book but showing how to stop a container and that the local port can be different than the contianer port.

```
$ docker stop cool_knuth
cool_knuth

$ docker container ls
CONTAINER ID  IMAGE       COMMAND     CREATED     STATUS      PORTS       NAMES

$ docker container run --rm -d -p 9000:8080 example/docker-node-hello:latest
417424e2609cf3a79aa9e1f4ab44e88bc6e8de470cdfc261a2f26e474fabf132

$ curl http://localhost:9000
Hello World. Wish you were here.
```