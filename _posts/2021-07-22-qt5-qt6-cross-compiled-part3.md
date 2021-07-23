---
title: Cross compile Qt 5.15.X and 6.X.X for arm architecture with tolchain created by crosstool-ng (Docker) - Part 2
category: Dev
tags: [toolchain, docker, crosstool-ng, crosstool-NG, cross, cross-compile, arm, gcc, g++, Qt, Qt6, Qt 6.2.0]
---
In this post we will see how to cross-compile Qt 6.2.0 with the toolchain we generated in part 1 of this series.

I recommend reading the following links for more details:

* [Qt6 Configure options](https://doc-snapshots.qt.io/qt6-dev/configure-options.html){:target="_blank"}
* [Qt 6 Build System](https://www.qt.io/blog/qt-6-build-system){:target="_blank"}

## Requirements

Basic requirements for this post:

* Have done the steps in part 1 of this post
* Have the Qt 6.2.0 source downloaded in the /opt/Qt/6.2.0 folder, which can be done via the Maintenance Tool or download from [git](https://wiki.qt.io/Building_Qt_6_from_Git){:target="_blank"}

> In this post I won't explain what Qt Lite is and what each configure option means. For this, read the **links mentioned above**. ![emoji](/assets/img/emoji/smirk.png)

## Starting

In the previous post we included the /opt/Qt folder as a volume in the Docker, we will make use of it now.

```bash
cd /opt/Qt/6.2.0
cp -r Src src-6.2.0
cd src-6.2.0
```

> With the above commands, we are copying the source downloaded by Maintenance Tool. If you have downloaded from the link and extracted, replace `Src` with the name of the unzipped folder.

Create a folder inside qtbase/mkspecs/devices called `linux-arm-pos-g++` and the following files within it

* qmake.conf

```conf
#
# qmake configuration for linux-g++ using linux-arm-pos-g++ compiler
#

MAKEFILE_GENERATOR      = UNIX
CONFIG                 += incremental
QMAKE_INCREMENTAL_STYLE = sublib

include(../common/linux.conf)
include(../common/gcc-base-unix.conf)
include(../common/g++-unix.conf)

# modifications to g++.conf
QMAKE_CC                = $${CROSS_COMPILE}gcc
QMAKE_CXX               = $${CROSS_COMPILE}g++
QMAKE_LINK              = $${CROSS_COMPILE}g++
QMAKE_LINK_SHLIB        = $${CROSS_COMPILE}g++

# modifications to linux.conf
QMAKE_AR                = $${CROSS_COMPILE}ar cqs
QMAKE_OBJCOPY           = $${CROSS_COMPILE}objcopy
QMAKE_NM                = $${CROSS_COMPILE}nm -P
QMAKE_STRIP             = $${CROSS_COMPILE}strip
load(qt_config)
```

* qplatformdefs.h

```cpp
#include "../../linux-g++/qplatformdefs.h"
```

For the Qt6 build we need to create a toolchain file for cmake. Create the following file in your toolchain:

```bash
mkdir -p /opt/toolchains/arm-verifone-linux-gnueabihf/libexec/cmake
vim /opt/toolchains/arm-verifone-linux-gnueabihf/libexec/cmake/toolchain.cmake
```

toolchain.cmake content file

```cmake
cmake_minimum_required(VERSION 3.15)
include_guard(GLOBAL)

set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR arm)

set(TARGET_SYSROOT /opt/toolchains/arm-verifone-linux-gnueabihf/arm-verifone-linux-gnueabihf/sysroot)
set(CROSS_COMPILER /opt/toolchains/arm-verifone-linux-gnueabihf/bin/arm-verifone-linux-gnueabihf)

set(CMAKE_SYSROOT ${TARGET_SYSROOT})

set(CMAKE_C_COMPILER ${CROSS_COMPILER}-gcc)
set(CMAKE_CXX_COMPILER ${CROSS_COMPILER}-g++)

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
```

Run the following commands on the root of the source

```bash
# export CROSS_COMPILE=[toolchain name]
export TOOLCHAIN=arm-verifone-linux-gnueabihf
export PATH="/opt/toolchains/${TOOLCHAIN}/bin:${PATH}"
export SYSROOT="/opt/toolchains/${TOOLCHAIN}/${TOOLCHAIN}/sysroot"
CMAKE_TOOLCHAIN=/opt/toolchains/arm-verifone-linux-gnueabihf/libexec/cmake/toolchain.cmake

cmake -B buildQt6 . -GNinja -DCMAKE_BUILD_TYPE=Release -DQT_HOST_PATH=/opt/Qt/6.2.0/gcc_64 -DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN} -DCMAKE_INSTALL_PREFIX=/opt/Qt/6.2.0/v240m -DCMAKE_STAGING_PREFIX=/opt/Qt/6.2.0/v240m -DQT_QMAKE_TARGET_MKSPEC=devices/linux-arm-pos-g++ -DQT_BUILD_EXAMPLES=FALSE -DQT_BUILD_TESTS=FALSE -DCMAKE_INTERPROCEDURAL_OPTIMIZATION_RELEASE=ON -DBUILD_WITH_PCH=OFF -DQT_QMAKE_DEVICE_OPTIONS=CROSS_COMPILE=arm-verifone-linux-gnueabihf- -DINPUT_reduce_exports=yes -DINPUT_optimize_size=yes -DBUILD_qt3d=OFF -DBUILD_qtcharts=OFF -DBUILD_qtconnectivity=OFF -DBUILD_qtdoc=OFF -DBUILD_qtlocation=OFF -DBUILD_qtlottie=OFF -DBUILD_qtmultimedia=OFF -DBUILD_qtquick3d=OFF -DBUILD_qtsensors=OFF -DBUILD_qtscxml=OFF -DBUILD_qtserialbus=OFF -DBUILD_qtserialport=OFF -DBUILD_qttools=OFF -DBUILD_qttranslations=OFF -DBUILD_qtwayland=OFF -DBUILD_qtwebengine=OFF -DBUILD_qtwebview=OFF -DBUILD_qtwebchannel=OFF -DBUILD_qtwebsockets=OFF -DBUILD_WITH_PCH=OFF -DINPUT_opengl=no -DBUILD_qtnetworkauth=OFF -DBUILD_qtopcua=OFF -DBUILD_qtmqtt=OFF -DBUILD_qtcoap=OFF -DBUILD_qtpositioning=OFF -DINPUT_widgets=no -DINPUT_use_gold_linker_alias=no -DINPUT_quickcontrols2_fusion=no -DINPUT_quickcontrols2_imagine=no -DINPUT_quickcontrols2_material=no -DINPUT_quickcontrols2_universal=no -DINPUT_textodfwriter=no -DINPUT_textmarkdownreader=no -DINPUT_textmarkdownwriter=no -DINPUT_testlib=no -DINPUT_vnc=no

cmake --build buildQt6 --parallel
cmake --install buildQt6
```

When finished, Qt will be installed in /opt/Qt/6.2.0/v240m.

In a next post we can see how to compile various libraries like OpenSSL, libicuuc and etc with the cross-compile toolchain. ![emoji](/assets/img/emoji/sunglasses.png)
