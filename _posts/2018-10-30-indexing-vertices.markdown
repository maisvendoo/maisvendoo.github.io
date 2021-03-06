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

Разберем этот код подробнее. Разумеется, первым делом, мы создаем массив из шести вершин

```cpp
osg::ref_ptr<osg::Vec3Array> vertices = new osg::Vec3Array(6);
(*vertices)[0].set( 0.0f,  0.0f,  1.0f);
(*vertices)[1].set(-0.5f, -0.5f,  0.0f);
(*vertices)[2].set( 0.5f, -0.5f,  0.0f);
(*vertices)[3].set( 0.5f,  0.5f,  0.0f);
(*vertices)[4].set(-0.5f,  0.5f,  0.0f);
(*vertices)[5].set( 0.0f,  0.0f, -1.0f);
```

Инициализируем каждую вершину непосредственно, обращаясь вектору её координат с использованием операции разыменования указателя и оператора operator[] (мы помним, что osg::Array аналогичен по своему устройству std::vector).

Теперь создаем грани в виде списка индексов вершин

```cpp
osg::ref_ptr<osg::DrawElementsUInt> indices = new osg::DrawElementsUInt(GL_TRIANGLES, 24);
(*indices)[ 0] = 0; (*indices)[ 1] = 1; (*indices)[ 2] = 2; // Грань 0
(*indices)[ 3] = 0; (*indices)[ 4] = 4; (*indices)[ 5] = 1; // Грань 1
(*indices)[ 6] = 4; (*indices)[ 7] = 5; (*indices)[ 8] = 1; // Грань 2
(*indices)[ 9] = 4; (*indices)[10] = 3; (*indices)[11] = 5; // Грань 3
(*indices)[12] = 3; (*indices)[13] = 2; (*indices)[14] = 5; // Грань 4
(*indices)[15] = 1; (*indices)[16] = 5; (*indices)[17] = 2; // Грань 5
(*indices)[18] = 3; (*indices)[19] = 0; (*indices)[20] = 2; // Грань 6
(*indices)[21] = 0; (*indices)[22] = 3; (*indices)[23] = 4; // Грань 7
```

Грани будут треугольными, их будет 8, а значит список индексов должен содержать 24 элемента. Индексы граней идут в этом массиве последовательно: например грань 0 образована вершинами 0, 1 и 2; грань 1 - вершинами 0, 4 и 1; грань 2 - вершинами 4, 5 и 1 и т.д. Вершины перечисляются в порядке следования против часовой стрелки, если смотреть на лицевую сторону грани (смотрим рисунок выше).

Дальнейшие шаги по созданию геометрии мы выполняли в предыдущих примерах. Единственное чего мы не делали - автоматическая генерация сглаженных (усредненных) нормалей, которую мы выполняем в данном примере вызовом

```cpp
osgUtil::SmoothingVisitor::smooth(*geom);
```

Действительно, если заданы вершины грани, то легко рассчитать нормаль к ней. В вершинах, в которых сходятся несколько граней расчитывается некая усредненная нормаль - нормали сходящихся граней складываются и полученная сумма снова нормируется. Эти операции (а так же многое другое!) может выполнить сам движок с помощью классов из библиотеки osgUtil. Поэтому в нашем примере в файл *.pro мы добавим указание компоновщику линковать нашу программу и с этой библиотекой

**octahedron.pro**
```cmake
CONFIG(debug, debug|release) {

    TARGET = $$join(TARGET,,,_d)
		.
		.
		.    
    LIBS += -L$$OSG_LIB_DIRECTORY -losgUtild

} else {
		.
		.
		.
    LIBS += -L$$OSG_LIB_DIRECTORY -losgUtil
}
```

В итоге мы получаем следующий результат

![](https://habrastorage.org/webt/wz/xv/bz/wzxvbzdz9uvaakqzgjl6h6en86g.png)

Чтобы понять как это работает, рассмотрим конвейер OpenGL

![](https://habrastorage.org/webt/7w/f0/mq/7wf0mqmabsa1ltwbzsgt189jsmy.png)

Механизм массивов вершин уменьшает число вызовов OpenGL. Он сохраняет данные о вершинах в памяти приложения, которая используется на стороне клиента. Конвейер OpenGL на серверной стороне получает доступ к различным массивам вершин. Как показано на схеме, OpenGL получает данные из буфера вершин на стороне клиента и упорядоченным образом, выполняет сборку примитивов. Так происходит обработка данных при использовании методов set*Array() класса osg::Geometry. Класс osg::DrawArrays проходит по этим массивам непосредственно и отрисовывает их.

При использвании osg::DrawElements* обеспечивается понижени размерности массивов вершин и уменьшается число вершин, передаваемых в конвейер. Массив индексов позволяет сформировать на стороне сервера кэш вершин. OpenGL читает данные о вершинах из кэша, вместо того, чтобы читать из из буфера вершин на стороне клиента. Это существенно увеличивает общую производительность рендеринга.
