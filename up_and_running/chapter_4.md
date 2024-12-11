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
- The order of commands in your Dockerfile can impact the speed of builds. Move the commands that change to the bottom of the Dockerfile. This will limit the number of var DEFAULT_WHO = "World";
steps that have to be performed on subsequent builds
- It is best to only run a single process inside a container rather that more than one. This makes the container much easier to scale horizontally.
    - For example, do not run a database and frontend application inside the same container
- If you want to ensure all future Docker builds never use cache, use the `--no-cache` argument on the build command line.
    - This is a good troubleshooting steps to take if the final artifact is not working exactly as you expect when all other code has not changed.
- ARG and ENV arguments are powerful in making generic and reusable containers
- Multistage builds are imports functions to use to make containers as small as possible
- Layer and directory caching are powerful tools to help speed up builds
- Directory caching is a great way to save on container space
- Multiarchitecture builds are a great way to build for multiple CPU architectures without needing to run more than one command.


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

## Build Arguments

If you recall in the Dockerfile in the `docker-node-hello` repo there are a couple lines that look like this.

```
ARG email="anna@example.com"
LABEL "maintainer"=$email
```

Pay particular attention to the `ARG` line. The `ARG` argument means that the value can be changed via a build argument when building a Docker container. Note `ARG` is configuring a value called `email`. We can also see that `email` is a variable on the `LABEL maintainer` line. This means the `maintainer` `LABEL` can be changed at build time.

We can see the current value of the maintainer `LABEL` in the docker container we built earlier like this.

Using docker the output will look similar to this:

```
$ docker inspect example/docker-node-hello:latest | grep maintainer
                    "maintainer": "anna@example.com",
               "maintainer": "anna@example.com",

```

Using podman the output looks like this:

```
$ docker inspect localhost/example/docker-node-hello:latest | grep maintainer
                    "maintainer": "anna@example.com",
               "maintainer": "anna@example.com",
```

Since `maintainer` is a build argument we can change it.

```
docker image build --build-arg email=me@example.com -t example/docker-node-hello:latest .
```

Checking the value of maintainer we now see the following:

```
$ docker inspect localhost/example/docker-node-hello:latest | grep maintainer
                    "maintainer": "me@example.com",
               "maintainer": "me@example.com",
                    "created_by": "/bin/sh -c #(nop) LABEL \"maintainer\"=$email",
```

This example is only showing a `LABEL` being changed but what a build argument means is that anything in the docker container could be made more generic and changed at build time.

## Environment Variables as configuration

We know that if a Dockerfile contains `ARG` lines it means we can change values at build time, it is als possible to change environment variable values are run time. These run time changes are managed by entries in a Dockerfile that use the `ENV` argument.

In the `index.js` code we see the following. Notice the `WHO` variable value can be set if the environment variable `WHO` is configured. If it is not not configured it falls back to the value `World`.

```
var DEFAULT_WHO = "World";

var WHO = process.env.WHO || DEFAULT_WHO;

// App
var app = express();
app.get('/', function (req, res) {
  res.send('Hello ' + WHO + '. Wish you were here.\n');
});
```

If you recall when we ran the container previously the output message when we used curl or your web browser was:

```
Hello World. Wish you were here.
```

Let's set the environment variable for `WHO` to be `Human`.

```
$ docker container run --env WHO="Human" --rm -d -p 9000:8080 example/docker-node-hello:latest

$ curl http://localhost:9000
Hello Human. Wish you were here.
```

There is real power here. What this means is that your code can stay generic and change its configuration via environment variables. One way this can be beneficial is in local development and testing. Configuration for a database could be setup in your code to use environment variables rather that hard coded values. This means the container used in production can also be used for local testing by only changing values of relevant variables at runtime.

## Custom Base Images

Choosing the base image for your container can have a large impact on the consistency and size of the resulting artifact. There are times that building your own base image is preferable over a third managed image. The primary reason is that you know exactly what dependencies you need to successfully run your applicaiton. Some third party images may have more software installed than necessary.

A docker container should be seen as an application not as a server. This means that `full size` images of Debian or Ubuntu are, in most cases, unnecessary. SSH or compilers are not necessry to exist in the final container image.

What this all boils down to is that we would most likely be starting off with a slim version of Debian or Alpine Linux to get just enough of what we need. This is primarily due to the reason that the artifacts we creates rely upon shared libraries. Our code may not compile down to a static binary.

As a side note, if your code can compile down to a static binary you may be able to use what is called a `SCRATCH` base image which under the covers is nothing. The resulting image would be just your application so the Docker container size is the size of your image.

**NOTE:** The book is a bit of out of date regarding DNS problems with Alpine Linux on page 61. Alpine Linux is based off of the musl libraries rather then glibc. There was a significant number of years where this was a problem when it comes to DNS resolution.

Jumping into the DNS world for a moment.

DNS primarily works on the UDP protocol but if a response is too large the DNS communication will transition to using TCP. In past years musl libraries did not support this transition to TCP thus any DNS responses that were above a certain size would get dropped on the floor and applications would throw DNS resolution errors.

The musl libraries have solved this problem and Alpine Linux is much more stable when it comes to being a standard for base images.

Back to the Docker world now.

## Storing Images

If you only ever plan on building and running images on one computer this section will not mean much to you but in nearly every case, this is not reality. The build and running of an image usually happens on separate systems which means there needs to be a central registry to store the resulting artifacts.

The images can be stored in either Public or Private registries.

### Public Registries

A public registry is exactly what is sounds like. Images stored on it can be shared to others on the internet. Depending on the registry provider, they may provide the ability for images to be private. Some very well known docker registries are:

- [Docker Hub](https://hub.docker.com)
- [Github Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-docker-registry)
- [Red Hat's Quay.io](https://quay.io)
- [Google's Artifact Registry](https://cloud.google.com/artifact-registry/docs)

While it may not be as well known as those listed above, [AWS also has their own registry](https://gallery.ecr.aws/). The AWS registry is called `Elastic Container Registry (ECR)`.

A benefit to use a managed registry like those above is that we do not have to manage the infrastructure needed to support the registry. A flipside to this is that we are impacted by internet latencies. The larger a docker image is means it can take longer to download if the registry is on the other side of the world. This is another reason to make docker images as small as possible.

### Private Registries

A private registry gives all the functionality of a public registry but we control the infrastructure. We ensure that only our network can reach the registry. This can provide a valuable security protection in the event secrets were stored either by accident or necessity in a required docker container.

Docker [provides their own container](https://hub.docker.com/_/registry) for hosting a private docker registry. For our needs we would be using private registries in AWS ECR so we don't have to manage our own infrastructure.

**IMPORTANT:** In the event that we need to spin up our registry, we must keep in mind that the infrastructure used to host the registry MUST be independent of all other infrastructure. I know this from experience. A previous job had a self managed Kubernetes (K8s) cluster which had been originally built using images hosted from Docker hub. Once the infrastructure was up and running for many months they wanted to host their own registry. It was running in the K8s cluster. Everything was fine until the day the cluster crashed and all services were down. As you can imagine it was very difficult trying to spin up a K8s cluster with no registry. The fix was to create an EC2 instance that only ran the docker registry.

**NOTE:** In that same job the managment of the cluster became a bit of pain when old images needed to be removed. That job used the Docker provided registry. Things may have changed in over 5 years but at the time the process to remove old images was as follows:

- Put the registry into maintenance mode which requires a service restart
    - Maintenenace mode means no images can be written
- Run a garbage collection process that could take several hours
- Disable maintenance mode and restart the service

### Authenticating to a Registry

Since our use case is pretty limited compared to the book, I will not be covering creating or authenticating to Docker Hub. However, the process of logging into AWS ECR will be covered since it is relevant to us.

Since we use private registries inside AWS it is necessary that we login with docker cli. The standard way using the AWS CLI is as follows:


```
aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com
```

There are two pieces of information that are necessary to be known.

- $REGION - would be something like us-west-2
- $AWS_ACCOUNT_ID - would be the numeric value of the AWS account the registry is stored.

In order to be able to push images to ECR the proper permissions are necessary. At minimum, the account logging in needs the following. These permissions allow for the listing, describing, downloading, and uploading of images from a specific registry.


```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ecr:BatchCheckLayerAvailability",
                "ecr:BatchGetImage",
                "ecr:CompleteLayerUpload",
                "ecr:DescribeImages",
                "ecr:DescribeImageScanFindings",
                "ecr:DescribeRepositories",
                "ecr:GetAuthorizationToken",
                "ecr:GetDownloadUrlForLayer",
                "ecr:GetLifecyclePolicy",
                "ecr:GetLifecyclePolicyPreview",
                "ecr:GetRepositoryPolicy",
                "ecr:InitiateLayerUpload",
                "ecr:ListImages",
                "ecr:ListTagsForResource",
                "ecr:PutImage",
                "ecr:UploadLayerPart",
            ],
            "Resource": "arn:aws:ecr:region:111122223333:repository/repository-name"
        },
        {
            "Effect": "Allow",
            "Action": "ecr:GetAuthorizationToken",
            "Resource": "*"
        }
    ]
}
```

### Pusing Images

Most of the information from the book is going to skipped in these notes since I just want to cover concepts.

A key take away with pushing an image is knowing that the tag of the image is what determines where the container will be pushed. If a fully qualified domain name (FQDN) is not provided the image will attempt to be uploaded to Docker Hub.

In order to push to AWS ECR the image must be tagged in the following way:

```
$AWS_ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/repository-name:tag
```

The tagging can be done in a couple of ways.

#### Tag At Build

In a previous example we saw the tag at build time.

```
docker image build -t example/docker-node-hello:latest .
```

For our purposes we would tag images for ECR like this.

```
docker image build -t $AWS_ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/repository-name:tag .
```

#### Tag After Build

There are two ways to tag an image after a build. You can either use the image hash or the existing tag on an image. The following example is using the image hash but the steps are exactly the same way with an existing image.

In a preivous example the end of the build looked like this.

```
Successfully tagged localhost/example/docker-node-hello:latest
f25dcfc6637690e93eb2711c54e8441ddd19aae4f575e057a0bae51f0805aca1
```

The docker container could be tagged thusly for AWS ECR:

```
docker image tag f25dcfc6637690e93eb2711c54e8441ddd19aae4f575e057a0bae51f0805aca1 $AWS_ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/repository-name:tag
```

Here is a more concrete example for our previous work.

List our existing docker images. There may be some differences here than what you see on your screen.

```
$ docker images
REPOSITORY                           TAG         IMAGE ID      CREATED        SIZE
localhost/example/docker-node-hello  latest      9b2490cea318  3 days ago     1.05 GB
<none>                               <none>      f25dcfc66376  6 days ago     1.05 GB
docker.io/library/node               18.13.0     b68a472583ef  23 months ago  1.02 GB
```

Notice the image ID for `localhost/example/docker-node-hello` is `9b2490cea318`. Let's say I want to tag that image for AWS ECR.

```
$ docker image tag 9b2490cea318 1234567890.dkr.ecr.us-west-2.amazonaws.com/docker-node-hello:latest

$ docker images
REPOSITORY                                                    TAG         IMAGE ID      CREATED        SIZE
localhost/example/docker-node-hello                           latest      9b2490cea318  3 days ago     1.05 GB
1234567890.dkr.ecr.us-west-2.amazonaws.com/docker-node-hello  latest      9b2490cea318  3 days ago     1.05 GB
<none>                                                        <none>      f25dcfc66376  6 days ago     1.05 GB
docker.io/library/node                                        18.13.0     b68a472583ef  23 months ago  1.02 GB
```

## Optimizing Images

Previously, it was stated that it is best to keep images as small as possible. The smaller an image is the faster it can be uploaded to a registry, the faster it can be downloaded from a registry, and the least amount of disk space is used up. It can also be a benefit from a security perspective because there is less software that has to be kept up to date.

### Keeping Images Small

We will take a look at the author provided Go application container.

```
docker container run --rm -d -p 8080:8080 spkane/scratch-helloworld
```

When using podman, the image needs to be fully qualified.

```
$ docker container run --rm -d -p 8080:8080 spkane/scratch-helloworld
Error: short-name "spkane/scratch-helloworld" did not resolve to an alias and no unqualified-search registries are defined in "/etc/containers/registries.conf"

$  docker container run --rm -d -p 8080:8080 docker.io/spkane/scratch-helloworld
Trying to pull docker.io/spkane/scratch-helloworld:latest...
Getting image source signatures
Copying blob 702599c1d55c done  
Copying config 8aa4ea322c done  
Writing manifest to image destination
Storing signatures
77c18466eb23b315d920445e9352d99e52c4074039c97dbf31eabb973e9c1180
```

We will verify the container is running.

```
$ curl http://localhost:8080
Hello World from Go in minimal Docker container
```

We can see the image size by looking at our images. The application size is 4.5 MB!

```
$ docker images
REPOSITORY                                                    TAG         IMAGE ID      CREATED        SIZE
localhost/example/docker-node-hello                           latest      9b2490cea318  3 days ago     1.05 GB
1234567890.dkr.ecr.us-west-2.amazonaws.com/docker-node-hello  latest      9b2490cea318  3 days ago     1.05 GB
<none>                                                        <none>      f25dcfc66376  7 days ago     1.05 GB
docker.io/spkane/scratch-helloworld                           latest      8aa4ea322c2b  18 months ago  4.57 MB <<<<
docker.io/library/node                                        18.13.0     b68a472583ef  23 months ago  1.02 GB
```

Let's go a bit deeper into the contents of the container. We need to know the contianer id. We can do this a couple of ways.

```
$ docker container ls -l
CONTAINER ID  IMAGE                                       COMMAND      CREATED        STATUS            PORTS                   NAMES
77c18466eb23  docker.io/spkane/scratch-helloworld:latest  /helloworld  2 minutes ago  Up 2 minutes ago  0.0.0.0:8080->8080/tcp  competent_swanson
```

or 

```
$ docker ps
CONTAINER ID  IMAGE                                       COMMAND      CREATED        STATUS            PORTS                   NAMES
77c18466eb23  docker.io/spkane/scratch-helloworld:latest  /helloworld  4 minutes ago  Up 4 minutes ago  0.0.0.0:8080->8080/tcp  competent_swanson
```

We can export the contents of the images.

```
$ docker container export 77c18466eb23 -o web-app.tar

$ ls -lh web-app.tar
-rw-r--r-- 1 mike mike 4.4M Dec  8 09:32 web-app.tar
```

Let's look at the contents of the exported tar file. We can see there is only one file in the container, `helloworld`, and it is a little over 4 MB in size. All the zero size files/directories must exist in every container and they are placed there at build time. You may see a file called `.dockerenv` rather than `.containerenv`. This is here as I am running podman.

```
$ tar -tvf web-app.tar 
drwxr-xr-t 0/0               0 2024-12-08 09:27 dev/
drwxr-xr-x 0/0               0 2024-12-08 09:27 etc/
-rwx------ 0/0               0 2024-12-08 09:27 etc/hostname
-rwx------ 0/0               0 2024-12-08 09:27 etc/hosts
lrwxrwxrwx 0/0               0 2024-12-08 09:27 etc/mtab -> /proc/mounts
-rwx------ 0/0               0 2024-12-08 09:27 etc/resolv.conf
-rwxr-xr-x 0/0         4562944 2023-05-19 16:20 helloworld
drwxr-xr-x 0/0               0 2024-12-08 09:27 proc/
drwxr-xr-x 0/0               0 2024-12-08 09:27 run/
-rwx------ 0/0               0 2024-12-08 09:27 run/.containerenv
drwxr-xr-x 0/0               0 2024-12-08 09:27 sys/
```

**Container Tangent**

Let's take a tangent here and actually run the binary inside the container.

```
$ mkdir hold

$ cd hold

$ tar xvf ../web-app.tar 
dev/
etc/
etc/hostname
etc/hosts
etc/mtab
etc/resolv.conf
helloworld
proc/
run/
run/.containerenv
sys/

$ ls
dev  etc  helloworld  proc  run  sys

$ file helloworld 
helloworld: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, Go BuildID=4GUXiaAyP7tVTJYbCemo/NQfK0DZLsTjKYIcZetKG/9CRkW306TL8WhmvPxCyl/i6Ercl4doZEMS-zwsi9-, stripped

$ ./helloworld 
Started, serving at 8080
panic: ListenAndServe: listen tcp :8080: bind: address already in use

goroutine 1 [running]:
main.main()
        /go/pkg/mod/github.com/spkane/scratch-helloworld@v0.0.0-20230113184334-a0a65c63cb29/helloworld.go:18 +0xeb

$ docker ps
CONTAINER ID  IMAGE                                       COMMAND      CREATED        STATUS            PORTS                   NAMES
77c18466eb23  docker.io/spkane/scratch-helloworld:latest  /helloworld  9 minutes ago  Up 9 minutes ago  0.0.0.0:8080->8080/tcp  competent_swanson

$ docker stop competent_swanson
competent_swanson

$ ./helloworld 
Started, serving at 8080
```

In another terminal window.

```
$ curl localhost:8080
Hello World from Go in minimal Docker container
```

**Back To The Book**

I am going to skip the investigation into the docker image and leave that as an exercise to the reader. The reason is that we skipped the steps of installing a Docker server and those steps are necessary for this section. While I believe these steps are useful to understand, I believe that trying to get an early understanding of Docker does not require this level of detail starting out.

### Multistage Builds

In traditional build processes, there may be artifacts/outputs generated during the compilation process that are not needed for the final application to run. For example, in a C program, the object file created during the linker operation can be thrown away once executable has been created. If we were creating a container for a compiled application the object file does not need to exist in the final container. A multistage build allows for the separation of compilation from application usage. You can copy the resulting application to a fresh container that does not contain any of the needed software to compile the application.

The example from the book covers a Go application. Go and Rust are applications that compile to static binaries which mean that shared libraries are not needed and the resulting image size can be very small.

Looking closely at the Dockerfile we see two distinct sections separated by the FROM lines.

```
# Build container
FROM docker.io/golang:alpine as builder
RUN apk update && \
    apk add git && \
    CGO_ENABLED=0 go install -a -ldflags '-s' \
    github.com/spkane/scratch-helloworld@latest

# Production container
FROM scratch
COPY --from=builder /go/bin/scratch-helloworld /helloworld
EXPOSE 8080
CMD ["/helloworld]
```

The compilation step install git which has dependencies as well as the source code needed for the hello world application. Notice the `as builder` text at the end of the first FROM line. This labeling allows later builds to copy files from that build step into a new build step. We can see that copy operation by looking at the following line:

```
COPY --from=builder /go/bin/scratch-helloworld /helloworld
```

The `/go/bin/scratch-helloworld` file path points to the location of the just built binary and the `/helloworld` path is to the final location in the new container.


When we build this container we see that quite a lot of software is installed that we don't need to run the final application.

```
$ docker image build -t scratch-go .
[1/2] STEP 1/2: FROM docker.io/golang:alpine AS builder
Trying to pull docker.io/library/golang:alpine...
Getting image source signatures
Copying blob 06f05ace1117 done  
Copying blob 34d97d9959dd done  
Copying blob 4f4fb700ef54 done  
Copying blob 38a8310d387e skipped: already exists  
Copying blob 3739efb98791 done  
Copying config 8a6d980675 done  
Writing manifest to image destination
Storing signatures
[1/2] STEP 2/2: RUN apk update &&     apk add git &&     CGO_ENABLED=0 go install -a -ldflags '-s'     github.com/spkane/scratch-helloworld@latest
fetch https://dl-cdn.alpinelinux.org/alpine/v3.21/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.21/community/x86_64/APKINDEX.tar.gz
v3.21.0-38-g75e920e2df8 [https://dl-cdn.alpinelinux.org/alpine/v3.21/main]
v3.21.0-65-g124a4988cc4 [https://dl-cdn.alpinelinux.org/alpine/v3.21/community]
OK: 25386 distinct packages available
(1/12) Installing brotli-libs (1.1.0-r2)
(2/12) Installing c-ares (1.34.3-r0)
(3/12) Installing libunistring (1.2-r0)
(4/12) Installing libidn2 (2.3.7-r0)
(5/12) Installing nghttp2-libs (1.64.0-r0)
(6/12) Installing libpsl (0.21.5-r3)
(7/12) Installing zstd-libs (1.5.6-r1)
(8/12) Installing libcurl (8.11.0-r2)
(9/12) Installing libexpat (2.6.4-r0)
(10/12) Installing pcre2 (10.43-r0)
(11/12) Installing git (2.47.1-r0)
(12/12) Installing git-init-template (2.47.1-r0)
Executing busybox-1.37.0-r8.trigger
OK: 19 MiB in 28 packages
go: downloading github.com/spkane/scratch-helloworld v0.0.0-20230113184334-a0a65c63cb29
--> e8352c4f546
[2/2] STEP 1/4: FROM scratch
[2/2] STEP 2/4: COPY --from=builder /go/bin/scratch-helloworld /helloworld
--> ec9b3ef90d1
[2/2] STEP 3/4: EXPOSE 8080
--> fee3d0d1244
[2/2] STEP 4/4: CMD ["/helloworld]
[2/2] COMMIT
--> 3a6b63c5ee5
3a6b63c5ee5d53b8ed49c3332e70c3ea3b163afba2e54fdce0e19b673002e295
```

If we look at the resulting images we can a stark difference in size.

```
$ docker images
localhost/scratch-go                                          latest      3a6b63c5ee5d  2 minutes ago  5.08 MB
<none>                                                        <none>      e8352c4f5467  2 minutes ago  351 MB
```

We built `localhost/scratch-go` which is 5 MB in size but the `<none>` container which does not have a tag because it is a part of the multistage build is 350 MB! Imagine that we did not have the multistage build. The final container would be 356 MB.

### Layers Are Additive

Understanding this concept is important in understanding how to make your container images as small as possible. If a layer in your Dockerfile installs a lot of dependencies or caches software via a RUN line you cannot remove that software if you run a second to perform a clean up on a second RUN line. This is because each layer is immutable. A later layer cannot delete files in a previous layer.

The book's example shows this clearly. The Dockerfile looks like this.

NOTE: When running the different build steps below, make sure to remove the resulting images to make sure you are not using any cached layers as the cached layers may still have larger layers than you expect.

```
FROM docker.io/fedora
RUN dnf install -y httpd
CMD ["/usr/sbin/htpd", "-DFOREGROUND"]
```

If we look at the history of the image we can see the size of each layer.

```
$ docker history fedora
ID            CREATED         CREATED BY                                     SIZE        COMMENT
2ab7add90b54  10 seconds ago  /bin/sh -c #(nop) CMD ["/usr/sbin/htpd", "...  0 B         FROM 2ab7add90b54
<missing>     12 seconds ago  /bin/sh -c dnf install -y httpd                168 MB      FROM docker.io/library/fedora:latest
aa6787b90fe6  5 weeks ago     CMD ["/bin/bash"]                              0 B         buildkit.dockerfile.v0
<missing>     5 weeks ago     ADD fedora-20241031.tar / # buildkit           163 MB      buildkit.dockerfile.v0
<missing>     5 weeks ago     ENV DISTTAG=f41container FGC=f41 FBR=f41       0 B         buildkit.dockerfile.v0
<missing>     5 weeks ago     LABEL maintainer=Clement Verna <cverna@fed...  0 B         buildkit.dockerfile.v0
```

We can see that the base Fedora image is 163 MB in size and the Apache web server is 168 MB in size. We know that the Apache web server itself is not that large but there are a lot of dependencies that get installed as well. By default, nealy all Linux distributions will cache downloaded programs to save on download bandwidth later. This is bad for us though.

Let's clean up that space.

```
FROM docker.io/fedora
RUN dnf install -y httpd
RUN dnf clean all
CMD ["/usr/sbin/htpd", "-DFOREGROUND"]
```

As the title of this section states, layers are additive.

```
$ docker history fedora
ID            CREATED        CREATED BY                                     SIZE        COMMENT
b0218b510314  4 seconds ago  /bin/sh -c #(nop) CMD ["/usr/sbin/htpd", "...  0 B         FROM b0218b510314
<missing>     6 seconds ago  /bin/sh -c dnf clean all                       790 kB      FROM 61310efbcecf
61310efbcecf  9 seconds ago  /bin/sh -c dnf install -y httpd                168 MB      FROM docker.io/library/fedora:latest
aa6787b90fe6  5 weeks ago    CMD ["/bin/bash"]                              0 B         buildkit.dockerfile.v0
<missing>     5 weeks ago    ADD fedora-20241031.tar / # buildkit           163 MB      buildkit.dockerfile.v0
<missing>     5 weeks ago    ENV DISTTAG=f41container FGC=f41 FBR=f41       0 B         buildkit.dockerfile.v0
<missing>     5 weeks ago    LABEL maintainer=Clement Verna <cverna@fed...  0 B         buildkit.dockerfile.v0
```

Since we know every command is a layer we need to perform the cleanup on the same line as the install of the Apache web server.

```
FROM docker.io/fedora
RUN dnf install -y httpd && dnf clean all
CMD ["/usr/sbin/htpd", "-DFOREGROUND"]
```

As we can see the Apache software layer is now 82.5 MB which means over 80 MB of space was taken up only for the caching of packages. This is uncesssary!

```
$ docker history fedora
ID            CREATED             CREATED BY                                     SIZE        COMMENT
7589543ab053  About a minute ago  /bin/sh -c #(nop) CMD ["/usr/sbin/htpd", "...  0 B         FROM 7589543ab053
<missing>     About a minute ago  /bin/sh -c dnf install -y httpd && dnf cle...  82.5 MB     FROM docker.io/library/fedora:latest
aa6787b90fe6  5 weeks ago         CMD ["/bin/bash"]                              0 B         buildkit.dockerfile.v0
<missing>     5 weeks ago         ADD fedora-20241031.tar / # buildkit           163 MB      buildkit.dockerfile.v0
<missing>     5 weeks ago         ENV DISTTAG=f41container FGC=f41 FBR=f41       0 B         buildkit.dockerfile.v0
<missing>     5 weeks ago         LABEL maintainer=Clement Verna <cverna@fed...  0 B         buildkit.dockerfile.v0
```

### Layer Caching

Since each command is a layer in a Dockerfile we can use this to our advantage. Docker will utilize this caching to speed up subsequent container builds. This can have a dramatic increase in speed. In my testing, using podman, I could not reproduce the results found in the book as changing the order of the COPY and RUN operations did not produce any significant difference. However, the concept is still a good idea from a structure standpoint. It is best to keep the operations that change very little at the top of the Dockerfile.

### Directory Caching

**NOTE:** Due to features not included in podman I had to spin up a VM to run native Docker.

In order to make use of directory caching an environment variable is needed to be set to tell [Docker to use BuildKit](https://docs.docker.com/build/buildkit/) which is an improved backend that provides many new features.

To use BuildKit, set the proper environment variable. You only need to type this once into the same terminal window you are running docker commands.

```
export DOCKER_BUILDKIT=1
```

This is the most relevant to use due to the extensive usage of npm. Being able to cache the directories used for installing software can greatly increase build times while not increasing image size since the cached directories are mounted in such a way as they do not exist in the final container image. The way this works is by bind mounting a directory in a specific way such that the directory for the dependencies only exists during the build. This means that the resulting container will not have any of the dependency software installed.

In building the open-mastermind application and looking at is resulting size we see it is 312 MB.

```
# docker image ls --format "{{ .Size }}" mastermind
312MB
```

Now we configure the Dockerfile to use bind mount for directory caching. There are two key items that matter here.

- The `# syntax=docker/dockerfile:1` line at the top tells docker to use new functionality.
- Update the pip RUN line with `--mount=type=cache,target=/root/.cache` which mounts a host directory into the container.

```
# syntax=docker/dockerfile:1    
FROM python:3.9.15-slim-bullseye        
RUN mkdir /app
WORKDIR /app       
COPY . /app                                                                      
RUN --mount=type=cache,target=/root/.cache pip install -r requirements.txt                  
WORKDIR /app/mastermind                          
CMD ["python", "mastermind.py"]
```

Build a brand new container without caching to make sure we populate the directory cache with all needed dependencies.

```
# time docker build --no-cache -t mastermind .
[+] Building 11.2s (14/14) FINISHED                                                                            docker:default
 => CACHED docker-image://docker.io/docker/dockerfile:1@sha256:db1ff77fb637a5955317c7a3a62540196396d565f3dd5742e76dddbb  0.0s
 => CACHED [stage-0 1/6] FROM docker.io/library/python:3.9.15-slim-bullseye@sha256:ffc6cb648d6993e7c90abb95c2481eb688a6  0.0s
 => [stage-0 2/6] RUN mkdir /app                                                                                         0.1s
 => [stage-0 3/6] WORKDIR /app                                                                                           0.0s
 => [stage-0 4/6] COPY . /app                                                                                            0.0s
 => [stage-0 5/6] RUN --mount=type=cache,target=/root/.cache pip install -r requirements.txt                             9.9s
 => [stage-0 6/6] WORKDIR /app/mastermind                                                                                0.0s 
real    0m 11.30s                                                                                                             
user    0m 0.09s
sys     0m 0.04s
```

Add `py-events` to the requirements.txt file and build again but this time using caching.

```
# time docker build -t mastermind .
[+] Building 0.5s (14/14) FINISHED                                                                             docker:default
 => CACHED docker-image://docker.io/docker/dockerfile:1@sha256:db1ff77fb637a5955317c7a3a62540196396d565f3dd5742e76dddbb  0.0s
 => CACHED [stage-0 2/6] RUN mkdir /app                                                                                  0.0s
 => CACHED [stage-0 3/6] WORKDIR /app                                                                                    0.0s
 => CACHED [stage-0 4/6] COPY . /app                                                                                     0.0s
 => CACHED [stage-0 5/6] RUN --mount=type=cache,target=/root/.cache pip install -r requirements.txt                      0.0s
 => CACHED [stage-0 6/6] WORKDIR /app/mastermind                                                                         0.0s
real    0m 0.61s
user    0m 0.04s
sys     0m 0.03s
```

We can see the image size got 35 MB smaller even though a dependency was added! This is impressive.

```
# docker image ls --format "{{ .Size }}" mastermind
277MB
```

## Troubleshooting Broken Builds

Being able to troubleshoot a broken build is a useful skill to be able to understand how to quickly solve a problem. A powerful feature of docker is that it is possible to start a shell into a container image layer.

First a broken Dockerfile needs to be created. The `RUN apt-get -y update` is updated to look like `RUN apt-get -y update-all`.

```
# Originally forked from: git@github.com:gasi/docker-node-hello.git
FROM docker.io/node:18.13.0

ARG email="anna@example.com"
LABEL "maintainer"=$email
LABEL "rating"="Five Stars" "class"="First Class"

USER root

ENV AP /data/app
ENV SCPATH /etc/supervisor/conf.d

RUN apt-get -y update-all

# The daemons
RUN apt-get -y install supervisor
RUN mkdir -p /var/log/supervisor

# Supervisor Configuration
COPY ./supervisord/conf.d/* $SCPATH/

# Application Code
COPY *.js* $AP/

WORKDIR $AP
RUN npm install
CMD ["supervisord", "-n"]
```

Attempt the build but disable the new BuildKit functionality so we can understand the old way of troubleshooting. To save space several lines of the build process are removed.

```
# DOCKER_BUILDKIT=0 docker image build -t hello --no-cache .
DEPRECATED: The legacy builder is deprecated and will be removed in a future release.
            BuildKit is currently disabled; enable it by removing the DOCKER_BUILDKIT=0
            environment-variable.

Sending build context to Docker daemon  9.728kB
Step 1/15 : FROM docker.io/node:18.13.0
18.13.0: Pulling from library/node
....
....
Step 7/15 : ENV SCPATH /etc/supervisor/conf.d
 ---> Running in 2410e2fdae61
 ---> Removed intermediate container 2410e2fdae61
 ---> 5d53038880f6
Step 8/15 : RUN apt-get -y update-all
 ---> Running in 407c13777499
E: Invalid operation update-all
The command '/bin/sh -c apt-get -y update-all' returned a non-zero code: 100
```

Something you do not get from this error is why the `apt-get` command failed. All we know is it failed. We need to know why. In order to interact with the container we need the last known good layer and in the example above its id is `5d53038880f6`.

```
# docker container run --rm -ti 5d53038880f6 /bin/bash
root@d632a82dd8ef:/# apt-get -y update-all
E: Invalid operation update-all
```

We can see that `upate-all` is not a valid operation for `apt-get`. We can run the proper command in the container to validate a fix before we update the Dockerfile.

```
# apt-get -y update
Get:1 http://deb.debian.org/debian bullseye InRelease [116 kB]
Get:2 http://deb.debian.org/debian-security bullseye-security InRelease [27.2 kB]
Get:3 http://deb.debian.org/debian bullseye-updates InRelease [44.1 kB]
Get:4 http://deb.debian.org/debian bullseye/main amd64 Packages [8066 kB]
Get:5 http://deb.debian.org/debian-security bullseye-security/main amd64 Packages [325 kB]
Get:6 http://deb.debian.org/debian bullseye-updates/main amd64 Packages [18.8 kB]
Fetched 8597 kB in 2s (5527 kB/s)                    
Reading package lists... Done
```

Revert the Dockfile back to a known good state but break the `npm install` line by updating it to be `npm installer`. Attempt another build but this time we want BuildKit to be enabled. Recall this was configured very early on when we did `export DOCKER_BUILDKIT=1`.

```
# docker image build -t hello --no-cache .
[+] Building 3.6s (12/12) FINISHED                                                                             docker:default
....
....
 => ERROR [8/8] RUN npm installer                                                                                        0.3s 
------                                                                                                                        
 > [8/8] RUN npm installer:                                                                                                   
0.330 Unknown command: "installer"                                                                                            
0.330 
0.330 Did you mean this?
0.330     npm install # Install a package
0.330 
0.331 To see a list of supported npm commands, run:
0.331   npm help
------
Dockerfile:28
--------------------
  26 |     WORKDIR $AP
  27 |     
  28 | >>> RUN npm installer
  29 |     
  30 |     CMD ["supervisord", "-n"]
--------------------
ERROR: failed to solve: process "/bin/sh -c npm installer" did not complete successfully: exit code: 1
```

The more modern way to attempt to troubleshoot is to use a `target` capbility of a multistage build to split out the parts of the build that work from the parts that don't. By doing this we know once inside the container the environment is configured exactly as necessary.

- Change `FROM docker.io/node:18.13.0` to `FROM docker.io/node:18.13.0 AS deploy`
- Add a line

```
WORKDIR $AP
    
FROM deploy # this line was added
RUN npm installer
```

We expect the build to fail but with the `AS deploy` addition and using BuildKit we can build only the first stage of the multistage build in order to troubleshoot the npm installer problem.

Notice the `--target deploy` addition to the build line.

```
# docker image build -t hello --no-cache --target deploy .
```

Just like we did with the old style troubleshooting attempt we can now start an interactive session with the new image that was built.

```
# docker container run --rm -ti hello /bin/bash
root@3d63f5f7fcad:/data/app# ls
index.js  package.json
root@3d63f5f7fcad:/data/app# npm install
npm WARN EBADENGINE Unsupported engine {
npm WARN EBADENGINE   package: 'formidable@1.0.13',
npm WARN EBADENGINE   required: { node: '<0.9.0' },
npm WARN EBADENGINE   current: { node: 'v18.13.0', npm: '8.19.3' }
npm WARN EBADENGINE }
npm WARN deprecated mkdirp@0.3.4: Legacy versions of mkdirp are no longer supported. Please update to mkdirp 1.x. (Note that the API surface has changed to use Promises in 1.x.)
npm WARN deprecated formidable@1.0.13: Please upgrade to latest, formidable@v2 or formidable@v3! Check these notes: https://bit.ly/2ZEqIau
npm WARN deprecated connect@2.7.9: connect 2.x series is deprecated

added 18 packages, and audited 19 packages in 986ms

8 vulnerabilities (1 low, 1 moderate, 6 high)

To address all issues (including breaking changes), run:
  npm audit fix --force

Run `npm audit` for details.
npm notice 
npm notice New major version of npm available! 8.19.3 -> 10.9.2
npm notice Changelog: https://github.com/npm/cli/releases/tag/v10.9.2
npm notice Run npm install -g npm@10.9.2 to update!
npm notice 
root@3d63f5f7fcad:/data/app# exit
exit
```

Either method is valid for troubleshooting and both are valuable tools in the troubleshooting toolbelt.

## Multiarchitecture Builds

When Docker was first released ARM was very early in the server space. This means that nearly every container was being built for x86. However, times have changed and ARM is becoming the dominant player for running containers. They can have a lower cost and have similar it not more performance than a comparable x86 CPU. Prior to multiarchitecture support in Docker it was necessary to have build servers of the different CPU types to build your different containers. This means your buildsystem had to be duplicated.

Enter Docker's multiarchitecture support which removes the need for duplicate build servers.

**NOTE:** You may need to configure your local Docker to support more than one architecture. This is what I ran into. I skipped the section talking about updating the Docker server since we are not running one. We only need to worry about our local builds.

```
# docker buildx build --platform linux/amd64,linux/arm64 -t wordchain .
[+] Building 0.0s (0/0)                                                                                        docker:default
ERROR: Multi-platform build is not supported for the docker driver.
Switch to a different driver, or turn on the containerd image store, and try again.
Learn more at https://docs.docker.com/go/build-multi-platform/

# docker buildx create \
>   --name container-builder \
>   --driver docker-container \
>   --bootstrap --use
[+] Building 12.2s (1/1) FINISHED                                                                                             
 => [internal] booting buildkit                                                                                         12.2s
 => => pulling image moby/buildkit:buildx-stable-1                                                                      11.9s
 => => creating container buildx_buildkit_container-builder0                                                             0.3s
container-builder
```

The way this multiarchitecture builds work is by Docker utilize virtualization under the covers to handle the different CPU types. The approach uses [QEMU](https://www.qemu.org/) and [binfmt_misc](https://docs.kernel.org/admin-guide/binfmt-misc.html).

Building for multiple architectures is straight forward in that it only requires using the `--platform` command line argument with a comma separated list of architectures.

```
# docker buildx build --platform linux/amd64,linux/arm64 -t wordchain .
[+] Building 160.3s (21/21) FINISHED                                                       docker-container:container-builder
 => [internal] load build definition from Dockerfile                                                                     0.0s
 => => transferring dockerfile: 460B                                                                                     0.0s
 => [linux/amd64 internal] load metadata for docker.io/library/alpine:3.15                                               1.3s
 => [linux/amd64 internal] load metadata for docker.io/library/golang:1.18-alpine3.15                                    1.4s
 => [linux/arm64 internal] load metadata for docker.io/library/golang:1.18-alpine3.15                                    1.4s
 => [linux/arm64 internal] load metadata for docker.io/library/alpine:3.15                                               1.3s
 => [internal] load .dockerignore                                                                                        0.0s
 => => transferring context: 93B                                                                                         0.0s
 => [linux/amd64 build 1/5] FROM docker.io/library/golang:1.18-alpine3.15@sha256:a94d2c2d687adc28a70ca4ae4ca06f6b4b17e  10.5s
 => [linux/arm64 deploy 1/3] FROM docker.io/library/alpine:3.15@sha256:19b4bcc4f60e99dd5ebdca0cbce22c503bbcff197549d7e1  0.7s
 => [linux/amd64 deploy 1/3] FROM docker.io/library/alpine:3.15@sha256:19b4bcc4f60e99dd5ebdca0cbce22c503bbcff197549d7e1  0.5s
 => [linux/arm64 build 1/5] FROM docker.io/library/golang:1.18-alpine3.15@sha256:a94d2c2d687adc28a70ca4ae4ca06f6b4b17e  12.8s
 => [linux/amd64 build 2/5] RUN apk --no-cache add     bash     gcc     musl-dev     openssl                             4.6s
 => [linux/arm64 build 2/5] RUN apk --no-cache add     bash     gcc     musl-dev     openssl                             5.9s
 => [linux/amd64 build 3/5] COPY . /build                                                                                0.1s
 => [linux/amd64 build 4/5] WORKDIR /build                                                                               0.0s
 => [linux/amd64 build 5/5] RUN go install github.com/markbates/pkger/cmd/pkger@latest &&     pkger -include /data/wor  56.2s
 => [linux/arm64 build 3/5] COPY . /build                                                                                0.1s
 => [linux/arm64 build 4/5] WORKDIR /build                                                                               0.0s
 => [linux/arm64 build 5/5] RUN go install github.com/markbates/pkger/cmd/pkger@latest &&     pkger -include /data/wo  139.9s
 => [linux/amd64 deploy 2/3] COPY --from=build /build/wordchain /                                                        0.0s
 => [linux/arm64 deploy 2/3] COPY --from=build /build/wordchain /                                                        0.0s
WARNING: No output specified with docker-container driver. Build result will only remain in the build cache. To push result image into registry use --push or to load image into docker use --load
```

Unfortunatley, we cannot see the result of our build due to not having the image pushed but we can look at the output of the container provided by the author. We can see that three different architectures are supported.

```
# docker manifest inspect docker.io/spkane/wordchain:latest
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.list.v2+json",
   "manifests": [
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 739,
         "digest": "sha256:4bd1971f2ed820b4f64ffda97707c27aac3e8eb773a64664126f64933dc5bfc0",
         "platform": {
            "architecture": "amd64",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 739,
         "digest": "sha256:10aba6975b124143b5812b5c1c2a2bd1250627dddf6989133468c359ce805084",
         "platform": {
            "architecture": "arm64",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 739,
         "digest": "sha256:133f276ffeafe382c443eb0981c626d3d91948b6bc625d4752ae1c13ca3b3fc8",
         "platform": {
            "architecture": "arm",
            "os": "linux",
            "variant": "v7"
         }
      }
   ]
}
```