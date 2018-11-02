---
layout: post
title:  "Введение в OpenSceneGraph: Ноды-переключатели"
date:   2018-11-02 22:58:00 +0300
categories: jekyll update
---

Рассмотрим еще один класс - osg::Switch, позволяющий отображать или пропускать рендеринг узла сцены, в зависимости от некоего логического условия. Он является наследником класса osg::Group и прикрепляет к кадой своей дочерней ноде некоторое логическое значение. Он имеет несколько полезных публичных методов:

1. Перегруженный addChild(), в качестве второго параметра принимающий логический ключ, указывающий отображать или нет данный узел.
2. setValue() - установка ключа видимости/невидимости. Принимает индекс интересующей нас дочерней ноды и желаемое значение ключа. Соответственно getValue() позволяет получить текущее значение ключа по индексу интересующей нас ноды.
3. setNewChildDefaultValue() - установка дефолтного занчения ключа видимости для всех новых объектов, добавляемых в качестве дочерних.

Рассмотрим применение данного класса на примере.

**main.h**
```cpp
#ifndef     MAIN_H
#define     MAIN_H

#include    <osg/Switch>
#include    <osgDB/ReadFile>
#include    <osgViewer/Viewer>

#endif
```

**main.cpp**
```cpp
#include    "main.h"

int main(int argc, char *argv[])
{
    (void) argc; (void) argv;

    osg::ref_ptr<osg::Node> model1 = osgDB::readNodeFile("../data/cessna.osg");
    osg::ref_ptr<osg::Node> model2 = osgDB::readNodeFile("../data/cessnafire.osg");

    osg::ref_ptr<osg::Switch> root = new osg::Switch;
    root->addChild(model1.get(), false);
    root->addChild(model2.get(), true);

    osgViewer::Viewer viewer;
    viewer.setSceneData(root.get());

    return viewer.run();
}
```

Пример тривиален - мы загружаем две модели: обычную цессну и цессну с эффектом горящего двигателя

```cpp
osg::ref_ptr<osg::Node> model1 = osgDB::readNodeFile("../data/cessna.osg");
osg::ref_ptr<osg::Node> model2 = osgDB::readNodeFile("../data/cessnafire.osg");
```

Однако, в качестве корневой ноды создаем osg::Switch, что позволяет нам, при добавлении в неё моделей в качестве дочерних узлов задать ключ видимости для каждой из них

```cpp
osg::ref_ptr<osg::Switch> root = new osg::Switch;
root->addChild(model1.get(), false);
root->addChild(model2.get(), true);
```

То есть, model1 не будет рендерится, а model2 будет, что мы и пронаблюдаем, запустив программу

![](https://habrastorage.org/webt/ne/ql/xu/neqlxu8zkqzvsusyzeoaqow8urg.png)

Поменяв местами значения ключей будем видеть противоположную картину

```cpp
root->addChild(model1.get(), true);
root->addChild(model2.get(), false);
```

![](https://habrastorage.org/webt/fs/ae/z4/fsaez4jingyv1u3lx9-4ihsr3zq.png)

Взведя оба ключа, увидим две модели одновременно

```cpp
root->addChild(model1.get(), true);
root->addChild(model2.get(), true);
```

![](https://habrastorage.org/webt/vl/7j/za/vl7jzamish8ype6md3lljfnqsp0.png)

Включать видимость и невидимость ноды, дочерней для osg::Switch можно прямо на ходу, используя метод setValue()

```cpp
switchNode->setValue(0, false);
switchNode->setValue(0, true);
switchNode->setValue(1, true);
switchNode->setValue(1, false);
```

