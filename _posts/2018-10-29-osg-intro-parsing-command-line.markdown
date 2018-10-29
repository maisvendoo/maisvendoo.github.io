---
layout: post
title:  "Введение в OpenSceneGraph: Обработка параметров командной строки"
date:   2018-10-29 19:57:00 +0300
categories: jekyll update
---

Параметры командной строки передаются через аргументы функции main(). В прошлых примерах мы тщательно помечали эти параметры как неиспользуемые, теперь же воспользуемся ими, чтобы сообщить нашей программе некоторые данные при её запуске.

В OSG есть встроенные средства разбора командной строки. Создадим следующий пример

**main.h**
```cpp
#ifndef     MAIN_H
#define     MAIN_H

#include    <osgDB/ReadFile>
#include    <osgViewer/Viewer>

#endif // MAIN_H
```

**main.cpp**
```cpp
#include    "main.h"

int main(int argc, char *argv[])
{
    osg::ArgumentParser args(&argc, argv);
    std::string filename;
    args.read("--model", filename);

    osg::ref_ptr<osg::Node> root = osgDB::readNodeFile(filename);
    osgViewer::Viewer viewer;
    viewer.setSceneData(root.get());

    return viewer.run();
}
```

Задаем параметры запуска программы в QtCreator

![](https://habrastorage.org/webt/41/le/8p/41le8pjjdv2q-in3idhgnpe6-ao.png)

![](https://habrastorage.org/webt/t-/_o/bb/t-_obbiwlzwuax1e336kqd6tosy.png)