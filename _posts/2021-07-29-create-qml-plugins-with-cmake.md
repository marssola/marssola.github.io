---
title: Creating C++ plugins for QML with CMake
category: Dev
tags: [C++, Qt, Qt6, Qt5, QML, plugins, cmake]
---
How to create C++ plugins for QML with CMake

## Why this post?

So far, Qt Creator allows you to create a new C++ plugin project for QML, but with qmake. This post will show you how to do it with CMake.

I recommend reading the following links for more details:

* [Overview - QML and C++ Integration](https://doc.qt.io/qt-6/qtqml-cppintegration-overview.html){:target="_blank"}
* [Integrating QML and C++](https://doc.qt.io/qt-6/qtqml-cppintegration-topic.html){:target="_blank"}

## Starting

In:

* **Welcome - Projects - New** or **File Menu - New File or Project**
* Choose Library, C++ Library
* Choose where the project will be created and project name: UserRegister
* In Define Build System
  * Build System: CMake
* In Define Project Details
  * Type: Shared Library
  * Class name: UserRegisterPlugin
  * Header file: userregister_plugin.h
  * Source file: userregister_plugin.cpp
  * Qt module: Core
* Kit Selection
  * Choose Qt version

Qt Creator creates a very basic project. We'll do some customizations.

Remove the `UserRegister_global.h` file

In CMakeLists.txt, it looked like this:

```cmake
cmake_minimum_required(VERSION 3.14)

project(Register VERSION 1.0.0 LANGUAGES CXX)

set(CMAKE_INCLUDE_CURRENT_DIR ON)
set(CMAKE_AUTOUIC ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(QT NAMES Qt6 Qt5 COMPONENTS Core Quick Qml REQUIRED)
find_package(Qt${QT_VERSION_MAJOR} COMPONENTS Core Quick QuickControls2 REQUIRED)

set(PROJECT_SOURCES
    ${CMAKE_CURRENT_SOURCE_DIR}/src/userregister_plugin.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/src/userregister_plugin.h
    ${CMAKE_CURRENT_SOURCE_DIR}/src/register.h
    ${CMAKE_CURRENT_SOURCE_DIR}/src/register.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/resources/qml/qmldir
)

add_library(${PROJECT_NAME}
    SHARED
        ${PROJECT_SOURCES}
)

target_compile_definitions(${PROJECT_NAME}
    PRIVATE $<$<OR:$<CONFIG:Debug>,$<CONFIG:RelWithDebInfo>>:QT_QML_DEBUG>
)

target_link_libraries(${PROJECT_NAME}
    PRIVATE
        Qt${QT_VERSION_MAJOR}::Core
        Qt${QT_VERSION_MAJOR}::Quick
        Qt${QT_VERSION_MAJOR}::QuickControls2
        Qt${QT_VERSION_MAJOR}::Qml
)

set(PLUGIN_PATH ${QT_DIR}/../../../qml/User/${PROJECT_NAME})

install(TARGETS ${PROJECT_NAME} DESTINATION ${PLUGIN_PATH})
install(FILES resources/qml/qmldir DESTINATION ${PLUGIN_PATH})
```

> Note the structure of the files and the `qmldir` file created.

Contents of qmldir file:

```qmldir
module User.Register
plugin Register
```

In the class created by Qt Creator, in my case MyPlugin, leave it as:

File: `userregister_plugin.h`

```cpp
#pragma once

#include <QQmlExtensionPlugin>

class UserRegisterPlugin : public QQmlExtensionPlugin
{
    Q_OBJECT
    Q_PLUGIN_METADATA(IID QQmlExtensionInterface_iid)

public:
    void registerTypes(const char* uri) override;
};
```

File: `userregister_plugin.cpp`

```cpp
#include "userregister_plugin.h"

void UserRegisterPlugin::registerTypes(const char* /*uri*/)
{
}
```

This is the basic structure of the files. But the plugin won't be recognized yet because it doesn't have any modules registered.

Create a class called `Register`

File: `register.h`

```cpp
#pragma once

#include <QObject>
#include <QString>

class Register : public QObject
{
    Q_OBJECT

    Q_PROPERTY(QString firstName READ getFirstName WRITE setFirstName NOTIFY firstNameChanged)
    Q_PROPERTY(QString lastName READ getLastName WRITE setLastName NOTIFY lastNameChanged)
    Q_PROPERTY(QString nickname READ getNickname WRITE setNickname NOTIFY nicknameChanged)

public:
    explicit Register(QObject* parent = nullptr);

    const QString& getFirstName() const;
    void setFirstName(const QString& newFirstName);

    const QString& getLastName() const;
    void setLastName(const QString& newLastName);

    const QString& getNickname() const;
    void setNickname(const QString& newNickname);

signals:
    void firstNameChanged();
    void lastNameChanged();
    void nicknameChanged();

private:
    QString m_firstName;
    QString m_lastName;
    QString m_nickname;
};
```

File: `register.cpp`

```cpp
#include "register.h"

Register::Register(QObject* parent) : QObject(parent) { }

const QString& Register::getFirstName() const { return m_firstName; }

void Register::setFirstName(const QString& newFirstName)
{
    if (m_firstName == newFirstName)
        return;
    m_firstName = newFirstName;
    emit firstNameChanged();
}

const QString& Register::getLastName() const { return m_lastName; }

void Register::setLastName(const QString& newLastName)
{
    if (m_lastName == newLastName)
        return;
    m_lastName = newLastName;
    emit lastNameChanged();
}

const QString& Register::getNickname() const { return m_nickname; }

void Register::setNickname(const QString& newNickname)
{
    if (m_nickname == newNickname)
        return;
    m_nickname = newNickname;
    emit nicknameChanged();
}
```

To register the class as a component for QML

File: `userregister_plugin.cpp`

```cpp
#include "userregister_plugin.h"

#include "register.h"
#include <QtQml>

void UserRegisterPlugin::registerTypes(const char* uri)
{
    qmlRegisterType<Register>(uri, 1, 0, "Register");
}
```

In **`Projects`** in the kit you are using -> **`build`** -> **`Build Steps`**, set the target **`install`**. Build the project.

To test it, create a qml file with the content below and let's run it with qmlscene

File: `/tmp/UserRegisterApp.qml`

```qml
import QtQuick
import QtQuick.Controls
import User.Register

ApplicationWindow {
    id: window

    height: 640
    width: 480
    visible: true

    title: qsTr("Test User Register")

    Register {
        id: register

        onFirstNameChanged: console.log("First name:", firstName)
        onLastNameChanged: console.log("Last name:", lastName)
        onNicknameChanged: console.log("Nickname:", nickname)
    }

    Column {
        anchors.centerIn: parent
        spacing: 10

        TextField {
            id: fieldFirstName

            placeholderText: qsTr("First Name")
        }

        TextField {
            id: fieldLastName

            placeholderText: qsTr("Last Name")
        }

        TextField {
            id: fieldNickname

            placeholderText: qsTr("Nickname")
        }

        Button {
            id: buttonRegister

            text: qsTr("Register")

            onClicked: {
                register.firstName = fieldFirstName.text
                register.lastName = fieldLastName.text
                register.nickname = fieldNickname.text

                fieldFirstName.text = ""
                fieldLastName.text = ""
                fieldNickname.text = ""
            }
        }
    }
}
```

Running

```bash
/path/to/Qt/bin/qmlscene /tmp/UserRegisterApp.qml
```

Ready! Now you can migrate your QML plugins to CMake! ![emoji](/assets/img/emoji/sunglasses.png)
