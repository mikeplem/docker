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

