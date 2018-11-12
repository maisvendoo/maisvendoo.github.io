---
layout: post
title:  "Введение в OpenSceneGraph: Динамическое создание геометрии"
date:   2018-11-11 13:27:00 +0300
categories: jekyll update
---

При обходе графа сцены OSG выполняет передачу данных в конвейер OpenGL, который выполняется в отдельном потоке. Этот поток должен быть синхронизирован с другими потоками обработки в каждом кадре. Невыполнеие этого требования может привести к тому, что метод frame() завершится раньше обработки данных геометрии. Это приведет к непредсказуемому поведению программы и сбоям. OSG предлагает решение этой проблеммы в виде методы setDataVariance() класса osg::Object, являющегося базовым для вех объектов сцены. Можно установить три режима обработки объектов

1. UNSPECIFIED (по-умолчанию) - OSG самостоятельно определяет порядко обработки объекта.
2. STATIC - объект низменяем и порядок его обработки не важен. Существенно ускоряет рендеринг.
3. DYNAMIC - объект должен быть обработан до начала отрисовки.

Эту настройку можно задать в любой момент вызовом

```cpp
node->setDataVariance( osg::Object::DYNAMIC );
```

## Реализуем морфинг-анимацию

Общепринятой является пракатика модификации геометрии "на лету", то есть изменение координат вершин, нормалей цветов и текстур динамически в каждом кадре, получая видоизменяемую геометрию. Такой прием называется морфинг-анимацией. В данном случае решающим является порядок обработки геометрии - все её изменения должны быть пересчитаны до того, как начнется отрисовка. Для иллюстрации этого приема немного изменим пример с цветным квадратом, заставив одну из его вершин вращатся вокруг оси X.

**main.h**
```cpp
#ifndef		MAIN_H
#define		MAIN_H

#include    <osg/Geometry>
#include    <osg/Geode>
#include    <osgViewer/Viewer>

#endif
```

**main.cpp**
```cpp
#include	"main.h"

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
osg::Geometry *createQuad()
{
    osg::ref_ptr<osg::Vec3Array> vertices = new osg::Vec3Array;
    vertices->push_back( osg::Vec3(0.0f, 0.0f, 0.0f) );
    vertices->push_back( osg::Vec3(1.0f, 0.0f, 0.0f) );
    vertices->push_back( osg::Vec3(1.0f, 0.0f, 1.0f) );
    vertices->push_back( osg::Vec3(0.0f, 0.0f, 1.0f) );

    osg::ref_ptr<osg::Vec3Array> normals = new osg::Vec3Array;
    normals->push_back( osg::Vec3(0.0f, -1.0f, 0.0f) );

    osg::ref_ptr<osg::Vec4Array> colors = new osg::Vec4Array;
    colors->push_back( osg::Vec4(1.0f, 0.0f, 0.0f, 1.0f) );
    colors->push_back( osg::Vec4(0.0f, 1.0f, 0.0f, 1.0f) );
    colors->push_back( osg::Vec4(0.0f, 0.0f, 1.0f, 1.0f) );
    colors->push_back( osg::Vec4(1.0f, 1.0f, 1.0f, 1.0f) );

    osg::ref_ptr<osg::Geometry> quad = new osg::Geometry;
    quad->setVertexArray(vertices.get());
    quad->setNormalArray(normals.get());
    quad->setNormalBinding(osg::Geometry::BIND_OVERALL);
    quad->setColorArray(colors.get());
    quad->setColorBinding(osg::Geometry::BIND_PER_VERTEX);
    quad->addPrimitiveSet(new osg::DrawArrays(GL_QUADS, 0, 4));

    return quad.release();
}

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
class DynamicQuadCallback : public osg::Drawable::UpdateCallback
{
public:

    virtual void update(osg::NodeVisitor *, osg::Drawable *drawable);
};

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
void DynamicQuadCallback::update(osg::NodeVisitor *, osg::Drawable *drawable)
{
    osg::Geometry *quad = static_cast<osg::Geometry *>(drawable);

    if (!quad)
        return;

    osg::Vec3Array *vertices = static_cast<osg::Vec3Array *>(quad->getVertexArray());

    if (!vertices)
        return;

    osg::Quat quat(osg::PI * 0.01, osg::X_AXIS);
    vertices->back() = quat * vertices->back();

    quad->dirtyDisplayList();
    quad->dirtyBound();
}

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
int main(int argc, char *argv[])
{
    (void) argc; (void) argv;

    osg::Geometry *quad = createQuad();
    quad->setDataVariance(osg::Object::DYNAMIC);
    quad->setUpdateCallback(new DynamicQuadCallback);

    osg::ref_ptr<osg::Geode> root = new osg::Geode;
    root->addDrawable(quad);

    osgViewer::Viewer viewer;
    viewer.setSceneData(root.get());
    
    return viewer.run();
}
```

Создание квадрата вынесем в отдельную функцию

```cpp
osg::Geometry *createQuad()
{
    osg::ref_ptr<osg::Vec3Array> vertices = new osg::Vec3Array;
    vertices->push_back( osg::Vec3(0.0f, 0.0f, 0.0f) );
    vertices->push_back( osg::Vec3(1.0f, 0.0f, 0.0f) );
    vertices->push_back( osg::Vec3(1.0f, 0.0f, 1.0f) );
    vertices->push_back( osg::Vec3(0.0f, 0.0f, 1.0f) );

    osg::ref_ptr<osg::Vec3Array> normals = new osg::Vec3Array;
    normals->push_back( osg::Vec3(0.0f, -1.0f, 0.0f) );

    osg::ref_ptr<osg::Vec4Array> colors = new osg::Vec4Array;
    colors->push_back( osg::Vec4(1.0f, 0.0f, 0.0f, 1.0f) );
    colors->push_back( osg::Vec4(0.0f, 1.0f, 0.0f, 1.0f) );
    colors->push_back( osg::Vec4(0.0f, 0.0f, 1.0f, 1.0f) );
    colors->push_back( osg::Vec4(1.0f, 1.0f, 1.0f, 1.0f) );

    osg::ref_ptr<osg::Geometry> quad = new osg::Geometry;
    quad->setVertexArray(vertices.get());
    quad->setNormalArray(normals.get());
    quad->setNormalBinding(osg::Geometry::BIND_OVERALL);
    quad->setColorArray(colors.get());
    quad->setColorBinding(osg::Geometry::BIND_PER_VERTEX);
    quad->addPrimitiveSet(new osg::DrawArrays(GL_QUADS, 0, 4));

    return quad.release();
}
```

описание которой, в принципе не требуется, так как подобные действия мы проделывали многократно. Для модификации вершин этого квадрата пишем класс DynamicQuadCallback, наследуя его от osg::Drawable::UpdateCallback

```cpp
class DynamicQuadCallback : public osg::Drawable::UpdateCallback
{
public:

    virtual void update(osg::NodeVisitor *, osg::Drawable *drawable);
};
```

переобпределяя в нем метод update()

```cpp
void DynamicQuadCallback::update(osg::NodeVisitor *, osg::Drawable *drawable)
{
    osg::Geometry *quad = static_cast<osg::Geometry *>(drawable);

    if (!quad)
        return;

    osg::Vec3Array *vertices = static_cast<osg::Vec3Array *>(quad->getVertexArray());

    if (!vertices)
        return;

    osg::Quat quat(osg::PI * 0.01, osg::X_AXIS);
    vertices->back() = quat * vertices->back();

    quad->dirtyDisplayList();
    quad->dirtyBound();
}
```

Здесь мы получаем указатель на объект геометрии

```cpp
osg::Geometry *quad = static_cast<osg::Geometry *>(drawable);
```

читаем из геометрии список вершин (вернее указатель на него)

```cpp
osg::Vec3Array *vertices = static_cast<osg::Vec3Array *>(quad->getVertexArray());
```

Для получения последнего элемента (последней вершины) в массиве класс osg::Array предоставляем метод back(). Для выполнеия поворота врешины относительно оси X вводим кватернион

```cpp
osg::Quat quat(osg::PI * 0.01, osg::X_AXIS);
```

то есть мы задали кватернион, реализующий поворот вокруг оси X на угол 0.01 * Pi. Поворачиваем вершину умножением кватерниона на вектор, задающий координаты вершины

```cpp
vertices->back() = quat * vertices->back();
```

Последние два вызова пересчитывают дисплейный список и габаритный параллелепипед для выидоизмененной геометрии

```cpp
quad->dirtyDisplayList();
quad->dirtyBound();
```

В теле функции main() мы создаем квадрат, устанавливаем для него динамический режим отрисовки и добавляем обратный вызов, модифицирующий геометрию

```cpp
osg::Geometry *quad = createQuad();
quad->setDataVariance(osg::Object::DYNAMIC);
quad->setUpdateCallback(new DynamicQuadCallback);
```

Создание корневого узла и запуск вьювера я оставлю без разбора, так как это мы уже проделывали не менее двадцати раз в разных вариантах. В итоге мы имеем простейфую морфинг-анимацию

![](https://habrastorage.org/webt/rk/oy/rf/rkoyrfpjkx_kan0puqvyj1ocxps.gif) 

А теперь попробуйте убрать (закомментировать) вызов setDataVariance(). Возможно мы и не увидим ничего криминального в этом случае - по-умалчанию OSG пытается автоматически определить когда следует обновлять данные о геометрии, пытаясь синхронизироваться с отрисовкой. Тогда попробуйте изменить режим с DYNAMIC на STATIC и будет видно, что изображение рендерится не плавно, с заметными рывками, в консоль сыпятся ошибки и предупреждения типа этого

```
Warning: detected OpenGL error 'invalid value' at after RenderBin::draw(..)
```

Если не выполнить метод dirtyDisplayList(), то OpenGL проигнорирует все изменения геометрии и для отрисовки будет использовать дисплейный список, созданный в самом начале, при создании квадрата. Удалите этот вызов, и увидите, что никакой анимации нет.

Без вызова метода dirtyBound() не будет произведен пересчет ограничивающего параллелепипеда и OSG будет неверно выполнять отсечение невидимых граней.