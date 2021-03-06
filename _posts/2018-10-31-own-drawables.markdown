---
layout: post
title:  "Введение в OpenSceneGraph: Определение собственных геометрических объектов"
date:   2018-10-31 15:58:00 +0300
categories: jekyll update
---

Существует два важных виртуальных метода, определенных в классе osg::Drawable

* computeBound() - вычисляет габаритный параллелепипед вокруг геометрии объекта
* drawImplementation() - позволяет отрисовывать геометрию, используя вызовы OSG и OpenGL

Создавая свой класс на основе osg::Drawable мы можем переопределить эти методы и добавить собственный код отрисовки в подходящее для этого место.

Метод computeBound() - константный, возвращает osg::BoundingBox - габаритный параллелепипед, охватывающий всю геометрию. Простейший способ создания такого параллелепипеда - сконструировать его, указав в качестве парраметров координаты противоположных углов

```cpp
osg::BoundingBox bb(osg::Vec3(0, 0, 0), osg::Vec3(1, 1, 1));
```

Важно помнить, что osg::BoundingBox не управляется через умные указатели и ни один из его не следует путать с osg::BoundingSphere, о которой мы поговорим когда-нибудь потом.

Метод drawImplementation() - константный и должен содержать вызовы, предназначенные для отрисовки некоторой геометрии. В качестве входного параметра он принимает osg::RenderInfo&, в котором передается информация из рендера OSG. Этот метод вызывается внутри метода draw() класса osg::Drawable. Этот метод вызывается один раз, после чего обработанная информация о построенной геометрии запоминается в виде дисплейного списка OpenGL и может быть использована на следующем кадре отрисовки. Для того, чтобы избежать использования дисплейных списков, достаточно отключить эту функцию вызовом

```cpp
drawable->setUseDisplayList( false );
```

В таком случае метод drawImplementation() будет вызываться на каждом кадре отрисовки. Это может быть полезно, если создаваемая нами геометрия изменяется, например анимирована.