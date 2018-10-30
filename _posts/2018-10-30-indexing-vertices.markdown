---
layout: post
title:  "Введение в OpenSceneGraph: Индексирование вершин примитивов"
date:   2018-10-30 21:52:00 +0300
categories: jekyll update
---

Класс osg::DrawArrays работает хорошо, когда чтение данных вершин происходит напрямую из массивов, без пропусков и скачков. Однако, это не столь эффективно, когда одна и та же вершина может принадлежать нескольким граням объекта. Рассмотрим пример

![](https://habrastorage.org/webt/yp/is/wm/ypiswmcxzm2uokv9xdreonovmf4.png)

Куб имеет восемь вершин. Однако, как видно из рисунка (смотрим на развертку куба на плоскость) некоторые вершины принадлежат более чем одной грани. ЕСли строить куб из 12-ти треугольных граней, то эти вершины будут повторятся, и вместо массива на 8 вершин мы получим массив на 36 вершин, большинство из которых на деле являются одной и той же вершиной!

В OSG существуют классы osg::DrawElementsUInt, osg::DrawElementsUByte и osg::DrawElementsUShort, которые используют в качестве данных массивы индексов вершин, призваны решить описанную проблему. Массивы индексов хранят индексы вершин примитивов, описывающих грани и другие элементы геометрии. При применении этих классов для куба достаточно хранить массив из восьми вершин, которые асоциируются с гранями через массивы индексов.

Классы типа osg::DrawElements* устроены так же как и стандартный класс std::vector. Для добавления индексов может быть использован такой код

```cpp
osg::ref_ptr<osg::DrawElementsUInt> de = new osg::DrawElementsUInt(GL_TRIANGLES);
de->push_back(0); de->push_back(1); de->push_back(2);
de->push_back(3); de->push_back(0); de->push_back(2); 
```

Данный код определяет переднюю грань куба, изображенного на рисунке. 

Рассмотрим еще один показательный пример - октаэдр

![](https://habrastorage.org/webt/is/cg/3r/iscg3rdq59wftnoy7h1iulgoczk.png)

Интересен он потому, что содержит всего шесть вершин, но каждая вершина входит аж в четыре треугольных грани! Мы можем создать массив на 24 вершины для отображения всех восьми граней с помощью osg::DrawArrays. Однако мы поступим иначе - вершины будем хранить в массиве из шести элементов, а грани сгенерируем используя класс osg::DrawElementsUInt.

**main.h**
```cpp
#ifndef     MAIN_H
#define     MAIN_H

#include    <osg/Geometry>
#include    <osg/Geode>
#include    <osgUtil/SmoothingVisitor>
#include    <osgViewer/Viewer>

#endif
```

**main.cpp**
```cpp
#include    "main.h"

int main(int argc, char *argv[])
{
    osg::ref_ptr<osg::Vec3Array> vertices = new osg::Vec3Array(6);
    (*vertices)[0].set( 0.0f,  0.0f,  1.0f);
    (*vertices)[1].set(-0.5f, -0.5f,  0.0f);
    (*vertices)[2].set( 0.5f, -0.5f,  0.0f);
    (*vertices)[3].set( 0.5f,  0.5f,  0.0f);
    (*vertices)[4].set(-0.5f,  0.5f,  0.0f);
    (*vertices)[5].set( 0.0f,  0.0f, -1.0f);

    osg::ref_ptr<osg::DrawElementsUInt> indices = new osg::DrawElementsUInt(GL_TRIANGLES, 24);
    (*indices)[ 0] = 0; (*indices)[ 1] = 1; (*indices)[ 2] = 2;
    (*indices)[ 3] = 0; (*indices)[ 4] = 4; (*indices)[ 5] = 1;
    (*indices)[ 6] = 4; (*indices)[ 7] = 5; (*indices)[ 8] = 1;
    (*indices)[ 9] = 4; (*indices)[10] = 3; (*indices)[11] = 5;
    (*indices)[12] = 3; (*indices)[13] = 2; (*indices)[14] = 5;
    (*indices)[15] = 1; (*indices)[16] = 5; (*indices)[17] = 2;
    (*indices)[18] = 3; (*indices)[19] = 0; (*indices)[20] = 2;
    (*indices)[21] = 0; (*indices)[22] = 3; (*indices)[23] = 4;

    osg::ref_ptr<osg::Geometry> geom = new osg::Geometry;
    geom->setVertexArray(vertices.get());
    geom->addPrimitiveSet(indices.get());
    osgUtil::SmoothingVisitor::smooth(*geom);

    osg::ref_ptr<osg::Geode> root = new osg::Geode;
    root->addDrawable(geom.get());

    osgViewer::Viewer viewer;
    viewer.setSceneData(root.get());

    return viewer.run();
}
```


![](https://habrastorage.org/webt/wz/xv/bz/wzxvbzdz9uvaakqzgjl6h6en86g.png)
