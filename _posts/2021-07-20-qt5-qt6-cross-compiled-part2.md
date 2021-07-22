---
title: Cross compile Qt 5.15.X and 6.X.X for arm architecture with tolchain created by crosstool-ng (Docker) - Part 2
---
In this post we will see how to cross-compile Qt 5.15.2 with the toolchain we generated in part 1 of this series.

I recommend reading the following links for more details:

* [Qt5 Configure options](https://doc.qt.io/qt-5/configure-options.html){:target="_blank"}
* [Qt embedded for Linux](https://doc.qt.io/qt-5/embedded-linux.html){:target="_blank"}
* [What is Qt Lite?](https://www.qt.io/blog/2017/05/31/qt-lite-qt-5-9-lts){:target="_blank"}
* [Qt Lite options](https://qtlite.com){:target="_blank"}

## Requirements

Basic requirements for this post:

* Have done the steps in part 1 of this post
* Have the Qt 5.15.2 source downloaded in the /opt/Qt/5.15.2 folder, which can be done via the Maintenance Tool or download from this [link](https://download.qt.io/archive/qt/5.15/5.15.2/single){:target="_blank"}

> In this post I won't explain what Qt Lite is and what each configure option means. For this, read the **links mentioned above**. ![emoji](/assets/img/emoji/smirk.png)

## Starting

In the previous post we included the /opt/Qt folder as a volume in the Docker, we will make use of it now.

```bash
cd /opt/Qt/5.15.2
cp -r Src src-5.15.2
cd src-5.15.2
```

> With the above commands, we are copying the source downloaded by Maintenance Tool. If you have downloaded from the link and extracted, replace `Src` with the name of the unzipped folder.

Create a folder inside qtbase/mkspecs/devices called `linux-arm-pos-g++` and the following files within it

* qmake.conf

```conf
#
# qmake configuration for linux-g++ using linux-arm-pos-g++ compiler
#

MAKEFILE_GENERATOR      = UNIX
CONFIG                 += incremental gdb_dwarf_index
QMAKE_INCREMENTAL_STYLE = sublib

include(../common/linux_device_pre.conf)

QT_QPA_DEFAULT_PLATFORM = linuxfb

include(../common/linux_device_post.conf)

load(qt_config)
```

* qplatformdefs.h

```cpp
#include "../../linux-g++/qplatformdefs.h"
```

Run the following commands on the root of the source

```bash
# export CROSS_COMPILE=[toolchain name]
export TOOLCHAIN=arm-verifone-linux-gnueabihf
export PATH="/opt/toolchains/${TOOLCHAIN}/bin:${PATH}"
export SYSROOT="/opt/toolchains/${TOOLCHAIN}/${TOOLCHAIN}/sysroot"

./configure -device linux-arm-pos-g++ -device-option CROSS_COMPILE=${TOOLCHAIN}- -sysroot ${SYSROOT} -opensource -confirm-license -reduce-exports -release -optimize-size -make libs -ltcg -no-cups -no-opengl -no-feature-xml -no-feature-widgets -no-pch -no-gcc-sysroot -no-use-gold-linker -nomake examples -nomake tests -nomake tools \
-skip qt3d -skip qtcanvas3d -skip qtcharts -skip qtconnectivity -skip qtdoc -skip qtdocgallery -skip qtgamepad -skip qtlocation -skip qtlottie -skip qtmultimedia -skip qtnetworkauth -skip qtquick3d -skip qtpurchasing -skip qtscript -skip qtsensors -skip qtscxml -skip qtserialbus -skip qtserialport -skip qtspeech -skip qttranslations -skip qttools -skip qtxmlpatterns -skip qttranslations -skip qtwayland -skip qtwebengine -skip qtwebview -skip qtwebchannel -skip qtwebglplugin -skip qtwebsockets \
-no-feature-accessibility-atspi-bridge -no-feature-android-style-assets -no-feature-angle -no-feature-angle_d3d11_qdtd -no-feature-appstore-compliant -no-feature-avx2 -no-feature-bearermanagement -no-feature-big_codecs -no-feature-dbus -no-feature-dbus-linked -no-feature-cssparser -no-feature-cupsjobwidget -no-feature-direct2d -no-feature-direct2d1_1 -no-feature-direct3d11 -no-feature-direct3d11_1 -no-feature-direct3d9 -no-feature-directfb -no-feature-directwrite -no-feature-directwrite1 -no-feature-directwrite2 -no-feature-drm_atomic -no-feature-debug_and_release -no-feature-desktopservices -no-feature-dxgi -no-feature-dxgi1_2 -no-feature-dxguid -no-feature-effects -no-feature-egl -no-feature-egl_x11 -no-feature-eglfs -no-feature-eglfs_brcm -no-feature-eglfs_egldevice -no-feature-eglfs_gbm -no-feature-eglfs_mali -no-feature-eglfs_openwfd -no-feature-eglfs_rcar -no-feature-eglfs_viv -no-feature-eglfs_viv_wl -no-feature-eglfs_vsp2 -no-feature-eglfs_x11 -no-feature-glibc -no-feature-gnu-libiconv -no-feature-gtk3 -no-feature-opengles2 -no-feature-opengles3 -no-feature-opengles31 -no-feature-opengles32 -no-feature-pdf -no-feature-pkg-config -no-feature-qml-debug -no-feature-quickcontrols2-fusion -no-feature-quickcontrols2-imagine -no-feature-quickcontrols2-material -no-feature-quickcontrols2-universal -no-feature-texthtmlparser -no-feature-textmarkdownreader -no-feature-textmarkdownwriter -no-feature-textodfwriter -no-feature-systemtrayicon -no-feature-testlib -no-feature-vnc -no-feature-tuiotouch -no-feature-wizard -no-feature-xcb -no-feature-xcb-egl-plugin -no-feature-xcb-glx -no-feature-xcb-glx-plugin -no-feature-xcb-sm -no-feature-xcb-xlib -no-feature-xkbcommon -no-feature-xlib \
-prefix /usr/local/qt5 -extprefix /opt/Qt/5.15.2/v240m --recheck-all

make -j$(nproc)  && make install
```

> With Qt Lite it is possible to have a very lean distribution passing as parameter `-no-feature-[feature name]`.
In this build we are disabling many features and compiling Qt with QML support only, hence `-no-feature-widgets`

When finished, Qt will be installed in /opt/Qt/5.15.2/v240m.

In the next post we will see how to compile Qt 6.2.0 alpha. ![emoji](/assets/img/emoji/sunglasses.png)
