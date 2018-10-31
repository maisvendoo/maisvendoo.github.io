---
layout: post
title:  "Введение в OpenSceneGraph: Техники обработки полигонов"
date:   2018-10-31 09:06:00 +0300
categories: jekyll update
---

OpenSceneGraph поддерживает различные техники обработки полигональной сетки объектов геометрии сцены. Эти методы препроцессинга, такие как редукция полигонов и тесселяция, часто используются для создания и оптимизации полигональных моделей. Они имеют простой интерфейс, но в процессе работу выполняют массу сложных вычислений и не очень подходят для выполнения "на лету".

К описанным техникам относятся:

1. osgUtil::Simplifier - уменьшение числа треугольников в геометрии. Публичный метод simplify() используется для упрощения геометрии моделей.
2. osgUtil::SootingVisitor - вычисление нормалей. Метод smooth() может быть использован для генерации сглаженных нормалей для модели, вместо самостоятельного их расчета и явного задания через массив нормалей.
3. osgUtil::TangentSpaceGenerator - генерация касательных базисных векторов для вершин модели. Задействуется с помощью метода generate() и сохраняет результат, возвращаемый методами getTangentArray(), getNormalArray() и getBinormalArray(). Эти результаты могут быть использованы для различный атрибутов вершин при написании шейдеров на GLSL.
4. osgUtil::Tesselator - выполняет тесселяцию полигональной сетки - разбиение сложных примитивов на последовательность простых (метод retesselatePolygons())
5. osgUtil::TriStripVisitor - конвертирует геометрическую поверхность в набор полос треугольных граней, что позволяет выполнять рендеринг с эффективным расходом памяти. Метод stripify() конвертирует набор примитивов модели в геометрию на базе набора GL_TRIANGLE_STRIP.

Все методы принимают геометрию объекта в качестве параметра, передаваемого по ссылке osg::Geometry&, например так

```cpp
osgUtil::TriStripVisitor tsv;
tsv.stripify(*geom);
```
где под geom понимается экземпляр геометрии, описываемый умным указателем.

Классы osg::Simplifier, osg::SmoothingVisitor и osg::TriStripVisitor могут работать непосредственно с узлами графа сцены, например

```cpp
osgUtil::TriStripVisitor tsv;
node->accept(tsv);
```

Метод accept() обрабатывает все дочерние узлы, до тех пор, пока указанная операция не будет применена ко всем оконечным нодам этой части дерева сцены, хранящимся в нодах типа osg::Geode.

Попробуем на практике технику тесселяции.

**main.h**
```cpp
#ifndef     MAIN_H
#define     MAIN_H

#include    <osg/Geometry>
#include    <osg/Geode>
#include    <osgUtil/Tessellator>
#include    <osgViewer/Viewer>

#endif
```

**main.cpp**
```cpp
#include    "main.h"

int main(int argc, char *argv[])
{
	/*
		Создаем фигуру вида
		
		-----
		|  _|
		| |_
		|   |
		-----
	*/
	
    osg::ref_ptr<osg::Vec3Array> vertices = new osg::Vec3Array;
    vertices->push_back( osg::Vec3(0.0f, 0.0f, 0.0f) ); // 0
    vertices->push_back( osg::Vec3(2.0f, 0.0f, 0.0f) ); // 1
    vertices->push_back( osg::Vec3(2.0f, 0.0f, 1.0f) ); // 2
    vertices->push_back( osg::Vec3(1.0f, 0.0f, 1.0f) ); // 3
    vertices->push_back( osg::Vec3(1.0f, 0.0f, 2.0f) ); // 4
    vertices->push_back( osg::Vec3(2.0f, 0.0f, 2.0f) ); // 5
    vertices->push_back( osg::Vec3(2.0f, 0.0f, 3.0f) ); // 6
    vertices->push_back( osg::Vec3(0.0f, 0.0f, 3.0f) ); // 7

    osg::ref_ptr<osg::Vec3Array> normals = new osg::Vec3Array;
    normals->push_back( osg::Vec3(0.0f, -1.0f, 0.0f) );

    osg::ref_ptr<osg::Geometry> geom = new osg::Geometry;
    geom->setVertexArray(vertices.get());
    geom->setNormalArray(normals.get());
    geom->setNormalBinding(osg::Geometry::BIND_OVERALL);
    geom->addPrimitiveSet(new osg::DrawArrays(GL_POLYGON, 0, 8));

    osg::ref_ptr<osg::Geode> root = new osg::Geode;
    root->addDrawable(geom.get());

    osgViewer::Viewer viewer;
    viewer.setSceneData(root.get());

    return viewer.run();
}
```

Исходя из пространсвенного положения вершин в данном примере, видно, что мы пытаемся создать невыпуклый многоугольник из восьми вершин, применяя генерацию одной грани типа GL_POLYGON. Сборка и выполнение этого примера показывают, что того результата, что мы ожидаем, не получается - пример отображается некорректно

![](https://habrastorage.org/webt/yb/d3/0f/ybd30fn9fa9duvxedfyb6igt3ui.png)

Для исправления этой проблемы, построенную геометрию, перед передачей её во вьювер, следует подвергнуть тесселяции

```cpp
osgUtil::Tessellator ts;
ts.retessellatePolygons(*geom);
```

после которой мы получим корректный результат

![](https://habrastorage.org/webt/p7/wj/fa/p7wjfanogrvkbdxksemk5li_vdi.png)

Как это работает? Невыпуклый многоугольник, без применения корректной тесселяции, не будет отображаться так, как мы этого ожидаем, так как OpenGL, стремясь к оптимизации производительности будет рассматривать его как простой, выпуклый полигон или просто игнорировать, что может давать совершенно неожиданные результаты.

Класс osgUtil::Tessellator использует алгоритмы для трансформацию выпуклого многоугольника в серию невыпуклых - в нашем случае он трансформирует геометрию в GL_TRIANGLE_STRIP.

![](https://habrastorage.org/webt/05/ho/do/05hodotfc4iltbsnlyevejuhvfw.png)

Этот класс может обрабатывать многоугольники с отверстиями и самопересекающиеся многоугольники. Через публичный метод setWindingType() можно определить различные правила обработки, такие как GLU_TESS_WINDING_ODD или GLU_TESS_WINDING_NONZERO, которые задают внутреннюю и внешнюю области сложного многоугольника.