---
title: Cross compile Qt 5.15.X and 6.X.X for arm architecture with tolchain created by crosstool-ng (Docker) - Part 1
category: Dev
tags: [toolchain, docker, crosstool-ng, crosstool-NG, cross, cross-compile, arm, gcc, g++]
---
In this post we will see how to create a toolchain for arm architecture (v6, v7, ...) And it can be used for any architecture supported by crosstool-ng using Docker.

I recommend reading the **[crosstool-ng](https://crosstool-ng.github.io/docs){:target="_blank"}** documentation for more details.

## Requirements

Basic requirements for this post:

* Any Linux distribution
* Docker installed and properly configured

>I won't cover Docker installation in this post, a quick search on **Google** will help you easily. ![emoji](/assets/img/emoji/smirk.png)

## Starting

To make it easier for other users to use, let's create HOME for the toolchain in **/opt** and change the folder owner to be our user.

```bash
sudo mkdir -p /opt/toolchains
sudo chown -R $USER:$USER /opt/toolchains
cd /opt/toolchains
```

Create the **Dockerfile** file with the following content:

```Dockerfile
FROM ubuntu:20.04
ENV TZ=America/Sao_Paulo
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

ARG USER
ARG UID
ARG GID

RUN apt-get update
RUN apt-get install --yes git build-essential gcc g++ gperf bison flex texinfo help2man make libncurses5-dev libisl-dev autoconf automake libtool libtool-bin gawk wget bzip2 xz-utils unzip patch curl libstdc++6 m4 binutils dh-autoreconf libcunit1-ncurses libexpat1-dev python-dev sudo zsh rsync vim cmake ninja-build libxkbcommon0 libgl1-mesa-dev libfontconfig1-dev libdbus-1-dev
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
docker create --name toolchain-qt -v /opt/toolchains:/opt/toolchains -v /opt/Qt:/opt/Qt -t toolchains:latest
```

>In the -v parameter, we are inserting the host's /opt/toolchains folder as the docker volume in /opt/toolchains. That way, everything done inside Docker will be exported to the host. _**We've also included the /opt/Qt folder where it should contain a host's Qt installation for the Qt6 build**_.

Run the created container!

```bash
docker start toolchain-qt
docker exec -it toolchain-qt zsh
```

>In this command, we are running the container and starting with the **ZSH** shell. If you want to use **Bash**, just replace the zsh parameter with bash.

When logging in for the first time with ZSH, I usually like to install the _**[oh my zsh](https://ohmyz.sh)**_ theme to make the shell more interesting.

```bash
sh -c "$(wget https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh -O -)"
```

Create toolchain build folder

```bash
mkdir -p /opt/toolchains/builds
cd /opt/toolchains/builds
```

To learn about the architectures that crosstool-ng supports, run:

```bash
ct-ng list-samples
```

We will use the sample arm-unknown-linux-gnueabi as a base. The toolchain we will create will be for a POS device (Point-of-Sale) **PAX S920**. The processor version of this device is armv6.

```bash
mkdir -p $HOME/src
mkdir arm-unknown-linux-gnueabi && cd arm-unknown-linux-gnueabi
ct-ng arm-unknown-linux-gnueabi
```

> With this command, we create a configuration file with basic instructions for the toolchain.

To configure crosstool-ng use the command:

```bash
ct-ng menuconfig
```

With the command above, we will change the following options:

* Paths and misc options:
  * [ * ] Use obsolete features
  * (\${CT_PREFIX:-/opt/toolchains}/\${CT_HOST:+HOST-\${CT_HOST}/}\${CT_TARGET}) Prefix directory
* Toolchain options
  * (pax) Tuple's vendor string
* Operating System
  * Version of linux (4.4.275)
* Binary utilities
  * Version of binutils (2.32)
  * Linkers to enable (ld)
* C-library
  * Version of glibc (2.15 (OBSOLETE))
* C compiler
  * Version of gcc (8.5.0)
* Debug facilities
  * [ * ] duma
  * [ * ] gdb
  * [ * ] ltrace
  * [ * ] strace

> It is very important to combine _**binutils**_ versions with _**GCC**_. For this, a quick search can help you find the most suitable combination.
**To avoid linking issues with libc, choose the same version included with the device in crosstool-ng.**

To create the toolchain, run:

```bash
ct-ng build.$(nproc)
```

When finished, we will have the toolchain installed in the /opt/toolchains/arm-pax-linux-gnueabi folder.

Now we are going to create a toolchain for another device. We will do it for **Verifone v240m**, which contains the armv7 processor which has some peculiarities in crosstool-ng, generating a little more difficulty.

```bash
cd /opt/toolchains/builds
mkdir -p arm-cortexa9_neon-linux-gnueabihf
cd arm-cortexa9_neon-linux-gnueabihf
ct-ng arm-cortexa9_neon-linux-gnueabihf
ct-ng menuconfig
```

With the command above, we will change the following options:

* Paths and misc options:
  * [ * ] Use obsolete features
  * [&nbsp;&nbsp;&nbsp;&nbsp;] Try features marked as EXPERIMENTAL
  * (\${CT_PREFIX:-/opt/toolchains}/\${CT_HOST:+HOST-\${CT_HOST}/}\${CT_TARGET}) Prefix directory
* Toolchain options
  * (verifone) Tuple's vendor string
* Operating System
  * Version of linux (4.4.275)
* Binary utilities
  * Version of binutils (2.32)
  * Linkers to enable (ld)
* C-library
  * Version of glibc (2.15 (OBSOLETE))
  * [&nbsp;&nbsp;&nbsp;&nbsp;] Build libidn add-on
  * [&nbsp;&nbsp;&nbsp;&nbsp;] Build and install locales
  * [&nbsp;&nbsp;&nbsp;&nbsp;] Enable -fcommon flag for older version of glibc when using GCC >=10
* C compiler
  * Version of gcc (8.5.0)
  * [ * ] Optimize gcc libs for size
  * < > Use sjlj for exceptions
* Debug facilities
  * [ * ] duma
  * [ * ] gdb
  * [ * ] ltrace
  * [ * ] strace

To create the toolchain, run:

```bash
ct-ng build.$(nproc)
```

When finished, we will have the toolchain installed in the /opt/toolchains/arm-verifone-linux-gnueabihf folder.

After compiling the toolchain, you need to enter write permission on the toolchain:

```bash
chmod -R +w /opt/toolchains/arm-verifone-linus-gnueabihf
```

In the next post we will see how to compile Qt 5.15.2 with the toolchain we just generated. ![emoji](/assets/img/emoji/sunglasses.png)
