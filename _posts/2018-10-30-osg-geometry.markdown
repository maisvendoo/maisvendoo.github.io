---
layout: post
title:  "Введение в OpenSceneGraph: Применение класса osg::Geometry" 
date:   2018-10-30 13:40:00 +0300
categories: jekyll update
---

Как говорилось выше, OSG поддерживает методику создания примитивов из массивов вершин и использует буфер вершин для ускорения рендеринга. Для управления этим процессом в движке предусмотрено два механизма.

## Хранение данных геометрии: класс osg::Array

Класс osg::Array, как и многие другие из рассмотренных выше, является классом абстрактным, от которого наследуются несколько потомков, предназначенных для хранения данных, передаваемых в функции OpenGL. Идеология работы с данным классом подобна классу std::vector из стандартной библиотеки C++. Следующий код иллюстрирует добавление вектора в массив вершин методом push_back()

```cpp
vertices->push_back(osg::Vec3(1.0f, 0.0f, 0.0f));
```

Массивы OSG выделяются в куче и управляются посредством умных указателей. Однако это не касается элементов массивов, таких как osg::Vec3 или osg::Vec2, которые могут быть созданы и на стеке.

Класс osg::Geometry является высокоуровневой оберткой над функциями OpenGL, работающими с массивами вершин. Он является производным от класса osg::Drawable и может быть без проблем добавлен в список объектов osg::Geode. Этот класс принимает вышеописанные массивы в качестве входных данных и использует их для генерации геометрии средствами OpenGL.

## Вершины и их атрибуты

Вершина является атомарной единицей примитивов геометрии. Она обладает рядом атрибутов, описывающих точку двух- или трехмерного пространства. К атрибутам относятся: положение, цвет, вектор-нормаль, текстурные координаты, координаты тумана и т.д. Вершина всегда должна иметь положение в пространстве, что касается других атрибутов, они могут присутствовать опционально. OpenGL поддерживает 16 базовых атрибутов вершины и может создавать разные массивы для их хранения. Все массивы атрибутов поддерживаются классом osg::Geometry и могут быть заданы методами вида set*Array().

**Атрибуты вершин в OpenSceneGraph**

|---------------------|-----------------------|------------------------|----------------------------|
| Атрибут             |Тип данных             |метод osg::Geometry     |Эквивалентный вызов OpenGL  |
|---------------------|-----------------------|------------------------|----------------------------|
|Положение            |3D-вектор              |setVertexArray()        |glVertexPointer()           |
|Нормаль              |3D-вектор              |setNormalArray()        |glNormalPointer()           |
|Цвет                 |4D-вектор              |setColorArray()         |glColorPointer()            |
|Вторчный цвет        |4D-вектор              |setSecondaryColorArray()|glSecondaryColorPointerEXT()|
|Координаты тумана    |Float                  |setFogCoordArray()      |glFogCoordPointerEXT()      |
|Текстурные координаты|2D или 3D-вектор       |setTexCoordArray()      |glTexCoordPointer()         |
|Прочие атрибуты      |Определен пользователем|setVertexArribArray()   |glVertexAttribPointerARB()  |

Обычно, в графической подсистеме OpenGL, вершина имеет восемь текстурных координат и три общих атрибута. В принципе, каждая вершина должна устанавливать свои атрибуты, что приводит к образованию нескольих массивов атрибутов одинакового размера - в противном случае несовпадение размеров массивов может привести к неопределенному поведению движка. OSG поддерживает различные методы связывания между собой атрибутов вершин, например

```cpp
geom->setColorBinding(osg::Geometry::BIND_PER_VERTEX);
```
означает, что каждая вершина и каждый цвет вершины взаимно-однозначно соотносятся друг с другом. Однако, если взглянуть на такой код

```cpp
geom->setColorBinding(osg::Geometry::BIND_OVERALL);
```
то он применяет один цвет ко всей геометрии. Аналогично могут быть настроены взаимосвязи между другими атрибутами путем вызова методов setNormalBinding(), setSecondaryColorBinding(), setFogCoordBinding() и setVertexAttribBinding().

## Наборы примитивов геометрии

Следующим шагом после определения массивов атрибутов вершин является описание того, как данные вершины будут обработаны рендером. Виртуальный класс osg::PrimitiveSet используется для управления геометрическими примитивами, генерируемыми рендером из набора вершин. Класс osg::Geometry предоставляет несколько методов для работы с наборами примитивов геометрии:

* addPrimitiveSet() - передает указатель на набор примитивов в объект osg::Geometry.
* removePrimitiveSet() - удаление набора примитивов. В качетве параметров принимает начальный индекс наборов и число наборов, которое следует удалить.
* getPrimitiveSet() - возвращает набор примитивов по индексу, переданному в качестве параметра.
* getNumPrimitiveSets() - возвращает общее число наборов примитивов, связанных с данной геометрией.

Класс osg::PrimitiveSet является абстрактным и не инстанцируется, но от него наследуются несколько производных классов, инкапсулирующих наборы примитивов, которыми оперирует OpenGL, такие как osg::DrawArrays и osg::DrawElementsUInt.

Класс osg::DrawArrays использует несколько последовательных елементов массива вершин для конструирования геометрического примитива. Он может быть создан и прикреплен к геометрии вызовом метода

```cpp
geom->addPrimitiveSet(new osg::DrawArrays(mode, first, count));
```

Первый параметр mode задает тип примитива, аналогичный соответсвующим типам примитивов OpenGL: GL_POINTS, GL_LINE_STRIP, GL_LINE_LOOP, GL_LINES, GL_TRIANGLE_STRIP, GL_TRIANGLE_FAN, GL_TRIANGLES, GL_QUAD_STRIP, GL_QUADS и GL_POLYGON.

Первый и второй параметр задают первый индекс в массиве вершин и число вершин, из которых следует генерировать геометрию. **Причем OSG не проверяет, достаточно ли указанного числа вершин для построения заданной режимом геометрии, что может приводить к краху приложения!**

## Пример - рисуем цветной квадрат

Реализуем всё вышеописанное в виде простого примера

**main.h**
```cpp
#ifndef     MAIN_H
#define     MAIN_H

#include    <osg/Geometry>
#include    <osg/Geode>
#include    <osgViewer/Viewer>

#endif // MAIN_H
```

**main.cpp**
```cpp
#include    "main.h"

int main(int argc, char *argv[])
{
    osg::ref_ptr<osg::Vec3Array> vertices = new osg::Vec3Array;
    vertices->push_back(osg::Vec3(0.0f, 0.0f, 0.0f));
    vertices->push_back(osg::Vec3(1.0f, 0.0f, 0.0f));
    vertices->push_back(osg::Vec3(1.0f, 0.0f, 1.0f));
    vertices->push_back(osg::Vec3(0.0f, 0.0f, 1.0f));

    osg::ref_ptr<osg::Vec3Array> normals = new osg::Vec3Array;
    normals->push_back(osg::Vec3(0.0f, -1.0f, 0.0f));

    osg::ref_ptr<osg::Vec4Array> colors = new osg::Vec4Array;
    colors->push_back(osg::Vec4(1.0f, 0.0f, 0.0f, 1.0f));
    colors->push_back(osg::Vec4(0.0f, 1.0f, 0.0f, 1.0f));
    colors->push_back(osg::Vec4(0.0f, 0.0f, 1.0f, 1.0f));
    colors->push_back(osg::Vec4(1.0f, 1.0f, 1.0f, 1.0f));

    osg::ref_ptr<osg::Geometry> quad = new osg::Geometry;
    quad->setVertexArray(vertices.get());

    quad->setNormalArray(normals.get());
    quad->setNormalBinding(osg::Geometry::BIND_OVERALL);

    quad->setColorArray(colors.get());
    quad->setColorBinding(osg::Geometry::BIND_PER_VERTEX);

    quad->addPrimitiveSet(new osg::DrawArrays(GL_QUADS, 0, 4));

    osg::ref_ptr<osg::Geode> root = new osg::Geode;
    root->addDrawable(quad.get());

    osgViewer::Viewer viewer;
    viewer.setSceneData(root.get());

    return viewer.run();
}
```

После компиляции и выполнения получим вот такую красоту

![](https://habrastorage.org/webt/vn/ps/xd/vnpsxdcpmd9uisdkmhhkkykib48.png)

Данный пример нуждается в пояснении. Итак, первым делом мы создаем массив вершин квадрата, в котором хранятся их координаты

```cpp
osg::ref_ptr<osg::Vec3Array> vertices = new osg::Vec3Array;
vertices->push_back(osg::Vec3(0.0f, 0.0f, 0.0f));
vertices->push_back(osg::Vec3(1.0f, 0.0f, 0.0f));
vertices->push_back(osg::Vec3(1.0f, 0.0f, 1.0f));
vertices->push_back(osg::Vec3(0.0f, 0.0f, 1.0f));
```

Далее задаем массив нормалей. В нашем простом случае нам не требуется создавать нормаль для каждой вершины - достаточно описать один единичный вектор, направленный перпендикулярно плоскости квадрата к камере

```cpp
osg::ref_ptr<osg::Vec3Array> normals = new osg::Vec3Array;
normals->push_back(osg::Vec3(0.0f, -1.0f, 0.0f));
```

Зададим цвет для каждой из вершин

```cpp
osg::ref_ptr<osg::Vec4Array> colors = new osg::Vec4Array;
colors->push_back(osg::Vec4(1.0f, 0.0f, 0.0f, 1.0f));
colors->push_back(osg::Vec4(0.0f, 1.0f, 0.0f, 1.0f));
colors->push_back(osg::Vec4(0.0f, 0.0f, 1.0f, 1.0f));
colors->push_back(osg::Vec4(1.0f, 1.0f, 1.0f, 1.0f));
```

Теперь создаем объект геометрии, где будет хранится описание нашего квадрата, которое будет обработано рендером. Передаем в эту геометрию массив вершин

```cpp
osg::ref_ptr<osg::Geometry> quad = new osg::Geometry;
quad->setVertexArray(vertices.get());
```

Передавая массив нормалей, сообщаем движку, что будет использована одна единственная нормаль для всех вершин, указанием метода связывания ("биндинга") нормалей BIND_OVAERALL

```cpp
quad->setNormalArray(normals.get());
quad->setNormalBinding(osg::Geometry::BIND_OVERALL);
```

Передавая цвета вершин, напротив, указываем, что каждой вершине будет соответствовать собственный цвет

```cpp
quad->setColorArray(colors.get());
quad->setColorBinding(osg::Geometry::BIND_PER_VERTEX);
```

Теперь создаем набор примитивов для геометрии. Указываем, что из массива вершин следует генерировать квадратные (GL_QUADS) грани, взяв в качестве первой вершины вершину с индексом 0, а общее число вершин будет равно 4

```cpp
quad->addPrimitiveSet(new osg::DrawArrays(GL_QUADS, 0, 4));
```

Ну а передачу геометрии и запуск рендера пояснять, думаю, не стоит

```cpp
osg::ref_ptr<osg::Geode> root = new osg::Geode;
root->addDrawable(quad.get());

osgViewer::Viewer viewer;
viewer.setSceneData(root.get());

return viewer.run();
```

Приведенный код эквивалентен следующей конструкции на чистом OpenGL

```cpp
static const GLfloat vertices[][3] = { … };
glEnableClientState( GL_VERTEX_ARRAY );
glVertexPointer( 4, GL_FLOAT, 0, vertices );
glDrawArrays( GL_QUADS, 0, 4 );
```

