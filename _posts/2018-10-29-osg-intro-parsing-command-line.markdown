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

Запустив программу на выполнение получаем результат (моделька грузовика [взята из того же OpenSceneGraph-Data](https://github.com/openscenegraph/OpenSceneGraph-Data.git)) 

![](https://habrastorage.org/webt/t-/_o/bb/t-_obbiwlzwuax1e336kqd6tosy.png)

Теперь разберем пример построчно

```cpp
osg::ArgumentParser args(&argc, argv);
```

создает экземпляр класса парсера командной строки osg::ArgumentParser. При создании конструктору класса передаются аргументы, принимаемые функцией main() от операционной системы.

```cpp
std::string filename;
args.read("--model", filename);
```

выполняем разбор аргументов, а именно ищем среди них ключ "--model", помещая его значение в строку filename. Таким образом, посредством этого ключа мы передаем в программу имя файла с трехмерной моделью. Далее мы загружаем эту модель и отображаем её

```cpp
osg::ref_ptr<osg::Node> root = osgDB::readNodeFile(filename);
osgViewer::Viewer viewer;
viewer.setSceneData(root.get());

return viewer.run();
```

