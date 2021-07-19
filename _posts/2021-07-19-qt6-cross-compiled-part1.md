---
title: Cross compile Qt6 for arm architecture with tolchain created by crosstool-ng (Docker) - Part 1
---
In this post we will see how to create a toolkit for arm architecture (v6, v7, ...) And it can be used for any architecture supported by crosstool-ng using Docker.

I recommend reading the **[crosstool-ng](https://crosstool-ng.github.io/docs)** documentation for more details.

## Requirements

Basic requirements for this post:

* Any Linux distribution
* Docker installed and properly configured

>I won't cover Docker installation in this post, a quick search on **Google** will help you easily.

## Starting

To make it easier for other users to use, let's create HOME for the toolset in **/opt** and change the folder owner to be our user.

```bash
sudo mkdir -p /opt/toolchains
sudo chown -R $USER:$USER /opt/toolchains
cd /opt/toolchains
```

Create the **Dockerfile** file with the following content:

```Dockerfile
FROM ubuntu:18.04

ARG USER
ARG UID
ARG GID

RUN apt-get update
RUN apt-get install --yes git build-essential gcc g++ gperf bison flex texinfo help2man make libncurses5-dev libisl-dev autoconf automake libtool libtool-bin gawk wget bzip2 xz-utils unzip patch curl libstdc++6 m4 binutils dh-autoreconf libcunit1-ncurses libexpat1-dev python-dev sudo zsh rsync vim
RUN apt-get clean --yes && rm -rf /var/lib/apt/lists/*

ARG CROSSTOOL_NG_VERSION=master

RUN cd /tmp && git clone --branch ${CROSSTOOL_NG_VERSION} --depth 1 https://github.com/crosstool-ng/crosstool-ng.git && cd crosstool-ng && ./bootstrap && ./configure && make -j$(nproc) && make install && cd .. && rm -rf crosstool-ng

RUN useradd -m ${USER} --uid=${UID} && echo "${USER}:${USER}" | chpasswd && adduser ${USER} sudo
RUN echo "Set disable_coredump false" >> /etc/sudo.conf
RUN echo "%sudo ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
RUN echo "%${USER} ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

USER ${UID}:${GID}
WORKDIR /home/${USER}
```

Let's build our docker image with the following command:

```bash
docker build --no-cache --build-arg USER=$(id -nu) --build-arg UID=$(id -u) --build-arg GID=$(id -g) -t toolchains .
```

Let's create the container to build the toolchain and Qt6 based on the Docker image we just created.

```bash
docker create --name toolchain-qt6 -v /opt/toolchains/toolchains:/opt/toolchains -t toolchains:latest
```

Run the created container!

```bash
docker exec -it toolchain-qt6 zsh
```

>In this command, we are running the container and starting with the **ZSH** shell. If you want to use **Bash**, just replace the zsh parameter with bash.

...
>This post is not finished yet.
