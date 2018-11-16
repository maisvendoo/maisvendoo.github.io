---
layout: post
title:  "Введение в OpenSceneGraph: Пример реализации собственного плагина"
date:   2018-11-16 11:13:00 +0300
categories: jekyll update
---

Никто не мешает нам придумать собственный формат хранения данных о трехмерной геометрии, и мы его придумаем

**piramide.pmd**
```
vertex:  1.0  1.0 0.0
vertex:  1.0 -1.0 0.0
vertex: -1.0 -1.0 0.0
vertex: -1.0  1.0 0.0
vertex:  0.0  0.0 2.0
face: 0 1 2 3
face: 0 3 4
face: 1 0 4
face: 2 1 4
face: 3 2 4
```

Здесь в начале файла идет список вершин с их координатами. Индексы вершин идут по порядку, начиная от нуля. После списка вершин идет список граней. Каждая грань задается перечнем индексов вершин, из которых она образована. Как видно ничего сложного. Задача - считать этот файл с диска и сформировать на его основе трехмерную геометрию.

## Настройка проекта плагина: особенности сценария сборки

Если раньше мы собирали приложения, то теперь нам предстоит написать динамическую библиотеку, да не просто библиотеку а плагин к OSG, удовлетворяющий определенным требованиям. Выполнять эти требования начнем со сценария сборки проекта, который будет выглядеть так

**plugin.pro**
```cmake
TEMPLATE = lib

CONFIG += plugin
CONFIG += no_plugin_name_prefix

TARGET = osgdb_pmd

win32-g++: TARGET = $$join(TARGET,,mingw_,)

win32 {

    OSG_LIB_DIRECTORY = $$(OSG_BIN_PATH)
    OSG_INCLUDE_DIRECTORY = $$(OSG_INCLUDE_PATH)
    DESTDIR = $$(OSG_PLUGINS_PATH)

    CONFIG(debug, debug|release) {

        TARGET = $$join(TARGET,,,d)

        LIBS += -L$$OSG_LIB_DIRECTORY -losgd
        LIBS += -L$$OSG_LIB_DIRECTORY -losgViewerd
        LIBS += -L$$OSG_LIB_DIRECTORY -losgDBd
        LIBS += -L$$OSG_LIB_DIRECTORY -lOpenThreadsd
        LIBS += -L$$OSG_LIB_DIRECTORY -losgUtild

    } else {

        LIBS += -L$$OSG_LIB_DIRECTORY -losg
        LIBS += -L$$OSG_LIB_DIRECTORY -losgViewer
        LIBS += -L$$OSG_LIB_DIRECTORY -losgDB
        LIBS += -L$$OSG_LIB_DIRECTORY -lOpenThreads
        LIBS += -L$$OSG_LIB_DIRECTORY -losgUtil

    }

    INCLUDEPATH += $$OSG_INCLUDE_DIRECTORY
}

unix {

    DESTDIR = /usr/lib/osgPlugins-3.7.0

    CONFIG(debug, debug|release) {

        TARGET = $$join(TARGET,,,d)

        LIBS += -losgd
        LIBS += -losgViewerd
        LIBS += -losgDBd
        LIBS += -lOpenThreadsd
        LIBS += -losgUtild

    } else {

        LIBS +=  -losg
        LIBS +=  -losgViewer
        LIBS +=  -losgDB
        LIBS +=  -lOpenThreads
        LIBS += -losgUtil
    }
}

INCLUDEPATH += ./include

HEADERS += $$files(./include/*.h)
SOURCES += $$files(./src/*.cpp)
```

Отдельные нюансы разберем подробнее

```cmake
TEMPLATE = lib
```

означает что мы будем собирать библиотеку. Чтобы не происходила генерация символических ссылок, с помощью которых в *nix системах разруливаются вопросы конфликта версий библиотек, указываем системе сборки, что данная библиотека будет плагином, то есть будет загружаться в память "на лету"

```cmake
CONFIG += plugin
```

Далее исключаем генерацию перфикса lib, который добалвяется при использовании компиляторов семейства gcc и учитывается рантаймовым окруженим при загрузке библиотеки

```cmake
CONFIG += no_plugin_name_prefix
```

Задаем имя файлу библиотеки

```cmake
TARGET = osgdb_pmd
```

где pmd - расширение файла изобретенного нами формата 3D-моделей. Далее обязательно указываем, что в случае сборки MinGW к именя обязательно добавляется префикс mingw_

```cmake
win32-g++: TARGET = $$join(TARGET,,mingw_,)
```

Указываем путь сборки библиотеки: для Windows

```cmake
DESTDIR = $$(OSG_PLUGINS_PATH)
```

для Linux

```cmake
DESTDIR = /usr/lib/osgPlugins-3.7.0
```

Для линукс, при таком указании пути (что несоммненно является костылем, но я пока не нашел другого решения) даем на указанную папку с плагинами OSG права на запись от обычного пользователя

```
# chmod 666 /usr/lib/osgPlugins-3.7.0
```

Все остальные настройки сборки аналогичны применявшимся при сборке примеров-приложений ранее.