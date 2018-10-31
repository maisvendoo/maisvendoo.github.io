---
layout: post
title:  "Введение в OpenSceneGraph: Перехват геометрических атрибутов"
date:   2018-10-31 12:28:00 +0300
categories: jekyll update
---

Класс osg::Geometry управляет множеством данных, описывающих вершины и отображает полигональную сетку с использованием упорядоченного набора примитивов. Однако, данный класс не имеет представления о таких элементах топологии модели как грани, ребра и соотношения между ними. Этот нюанс мешает реализации таких вещей как перемещение определенных граней, например при анимировании моделей. В настоящее время OSG не поддерживает такую функциональность.

Однако в движке реализован ряд функторов, позволяющих перечитать атрибуты геометрии любого объекта и использовать их в целях моделирования топологии полигональной сетки. В C++ функтором называется конструкция, позволяющая использовать объект как функцию. 

Класс osg::Drawable предосталяет разработчику четыре типа функторов

1. osg::Drawable::AttributeFunctor - читает атрибуты вершин как массив указателей. Он имеет ряд виртуальных методов для применения атрибутов вершин разных типов данных. Для использования этого функтора необходимо описать класс и переопределить один или более его методов, внутри которых выполняются требуемые разработчику действия

```cpp
virtual void apply( osg::Drawable::AttributeType type, 
					unsigned int size, osg::Vec3* ptr )
{
	// Читаем 3-векторы в буффер с указателем ptr.
	// Первый параметр определяет тип атрибута
}
```

2. osg::Drawable::ConstAttributeFunctor - read-only версия предыдущего функтора: указатель на массив векторов передается как константный параметр
3. osg::PrimitiveFunctor - имитирует процесс рендеринга объектов OpenGL. Под видом рендеринга объекта производится вызов переопрелеенных разработчиком методов функтора. Этот функтор имеет два важных шаблонных подкласса: osg::TemplatePrimitiveFunctor<> и osg::TriangleFunctor<>. Эти классы получают в качестве параметров вершины примитива и передают их в пользовательские методы с применением оператора operator().
4. osg::PrimitiveIndexFunctor - выполняет те же действия, что и предыдущий функтор, но в качестве параметра принимает индексы вершин примитива.

Классы, производные от osg::Drawable, такие как osg::ShapeDrawable и osg::Geometry имеют метод accept() позволяющий применить различные функторы.

## Пример использования функтора примитивов

Проилюстрируем описанный функционал, на примере сбора информации о треугольных гранях и точках некоторой, определенной нами заранее геометрии.

**main.h**
```cpp
#ifndef     MAIN_H
#define     MAIN_H

#include    <osg/Geode>
#include    <osg/Geometry>
#include    <osg/TriangleFunctor>
#include    <osgViewer/Viewer>

#include    <iostream>

#endif
```

**main.cpp**
```cpp
#include    "main.h"

std::string vec2str(const osg::Vec3 &v)
{
    std::string tmp = std::to_string(v.x());
    tmp += " ";
    tmp += std::to_string(v.y());
    tmp += " ";
    tmp += std::to_string(v.z());

    return tmp;
}

struct FaceCollector
{
    void operator()(const osg::Vec3 &v1,
                    const osg::Vec3 &v2,
                    const osg::Vec3 &v3)
    {
        std::cout << "Face vertices: "
                  << vec2str(v1)
                  << "; " << vec2str(v2)
                  << "; " << vec2str(v3) << std::endl;
    }
};

int main(int argc, char *argv[])
{
    (void) argc; (void) argv;

    osg::ref_ptr<osg::Vec3Array> vertices = new osg::Vec3Array;
    vertices->push_back( osg::Vec3(0.0f, 0.0f, 0.0f) );
    vertices->push_back( osg::Vec3(0.0f, 0.0f, 1.0f) );
    vertices->push_back( osg::Vec3(1.0f, 0.0f, 0.0f) );
    vertices->push_back( osg::Vec3(1.0f, 0.0f, 1.5f) );
    vertices->push_back( osg::Vec3(2.0f, 0.0f, 0.0f) );
    vertices->push_back( osg::Vec3(2.0f, 0.0f, 1.0f) );
    vertices->push_back( osg::Vec3(3.0f, 0.0f, 0.0f) );
    vertices->push_back( osg::Vec3(3.0f, 0.0f, 1.5f) );
    vertices->push_back( osg::Vec3(4.0f, 0.0f, 0.0f) );
    vertices->push_back( osg::Vec3(4.0f, 0.0f, 1.0f) );

    osg::ref_ptr<osg::Vec3Array> normals = new osg::Vec3Array;
    normals->push_back( osg::Vec3(0.0f, -1.0f, 0.0f) );

    osg::ref_ptr<osg::Geometry> geom = new osg::Geometry;
    geom->setVertexArray(vertices.get());
    geom->setNormalArray(normals.get());
    geom->setNormalBinding(osg::Geometry::BIND_OVERALL);
    geom->addPrimitiveSet(new osg::DrawArrays(GL_QUAD_STRIP, 0, 10));

    osg::ref_ptr<osg::Geode> root = new osg::Geode;
    root->addDrawable(geom.get());

    osgViewer::Viewer viewer;
    viewer.setSceneData(root.get());

    osg::TriangleFunctor<FaceCollector> functor;
    geom->accept(functor);

    return viewer.run();
}
```

Опуская рассмотренный нами многократно процесс создания геометрии обратим внимание на следующее. Мы определяем структуру FaceCollector, для которой переопределяем оператор operator() следующим образом

```cpp
struct FaceCollector
{
    void operator()(const osg::Vec3 &v1,
                    const osg::Vec3 &v2,
                    const osg::Vec3 &v3)
    {
        std::cout << "Face vertices: "
                  << vec2str(v1)
                  << "; " << vec2str(v2)
                  << "; " << vec2str(v3) << std::endl;
    }
};
```
Данный оператор, при вызове будет выводить на экран координаты трех вершин, передаваемых ему движком. Функция vec2str необходима для перевода компонент вектора osg::Vec3 в std::string. Для вызова функтора создадим его экземпляр и передадим его объекту геометрии через метод accept()

```cpp
osg::TriangleFunctor<FaceCollector> functor;
geom->accept(functor);
```
Данный вызов, как говорилось выше, имитирует отрисовку геометрии, подменяя саму отрисовку вызовом переопределенного метода функтора. В данном случае он будет вызываться при "отрисовке" каждого из треугольников, из которых составлена геометрия примера. 

На экране мы получим такую геометрию

![](https://habrastorage.org/webt/ai/-a/6v/ai-a6vj4eeubetgk0bpicmjbgfm.png)

и такой выхлоп в консоль

```
Face vertices: 0.000000 0.000000 0.000000; 0.000000 0.000000 1.000000; 1.000000 0.000000 0.000000
Face vertices: 0.000000 0.000000 1.000000; 1.000000 0.000000 1.500000; 1.000000 0.000000 0.000000
Face vertices: 1.000000 0.000000 0.000000; 1.000000 0.000000 1.500000; 2.000000 0.000000 0.000000
Face vertices: 1.000000 0.000000 1.500000; 2.000000 0.000000 1.000000; 2.000000 0.000000 0.000000
Face vertices: 2.000000 0.000000 0.000000; 2.000000 0.000000 1.000000; 3.000000 0.000000 0.000000
Face vertices: 2.000000 0.000000 1.000000; 3.000000 0.000000 1.500000; 3.000000 0.000000 0.000000
Face vertices: 3.000000 0.000000 0.000000; 3.000000 0.000000 1.500000; 4.000000 0.000000 0.000000
Face vertices: 3.000000 0.000000 1.500000; 4.000000 0.000000 1.000000; 4.000000 0.000000 0.000000
```

Фактически, при вызове geom->accept(...) отрисовки треугольников не происходит, вызовы OpenGL имитируются, а вместо них выводятся данные о вершинах треугольника, отрисовка которого имитируется

![](https://habrastorage.org/webt/gx/gp/il/gxgpilewn7pw26w9tdy-e28ypds.png)

Класс osg::TemplatePrimitiveFunctor<T> собирает данные не только о треугольниках, но и о любых других примитивах OpenGL. Для реализации обработки этих данных необходимо переопределить следующие операторы в аргументе шаблона

```cpp
// Для точек
void operator()( const osg::Vec3&, bool );
// Для линий
void operator()( const osg::Vec3&, const osg::Vec3&, bool );
// Для треугольников
void operator()( const osg::Vec3&, const osg::Vec3&, const osg::Vec3&, bool );
// Для четырехугольников
void operator()( const osg::Vec3&, const osg::Vec3&, const osg::Vec3&, const osg::Vec3&, bool );
```


