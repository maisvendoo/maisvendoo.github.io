---
layout: post
title:  "Введение в OpenSceneGraph: Изменение глобальных настроек дисплея. Сглаживание"
date:   2018-11-06 01:23:00 +0300
categories: jekyll update
---

OSG позволяет разработчику управлять глобальными настройками отображения, на сонове которых работают камеры, вьюверы и рендерятся элементы сцены. Для этого импользуется паттерн синглтон, то есть юникальный объект, содержащий эти настройки, реализованный в виде класса osg::DisplaySettings. Следовательно, из нашего приложения, мы можем изменить эти настройки в любой момент

```cpp
osg::DisplaySettings *ds = osg::DisplaySettings::instance();
```

Синглтон osg::DisplaySettings содержит настройки, применяемые к вновь создаваемым устройствам рендеринга, контексту OpenGL графического окна. Можно варьировать следующие параметры:

1. setDoubleBuffer() - включение/выключение двойной буферизации. По-умолчанию включено.
2. setDepthBuffer() - влючить/выключить буффер глубины. По-умолчанию включен.
3. Установить разрядность альфа-буфера (alpha buffer), буфера трафарета (stencil buffer), накопительного буфера (accumulation buffer) птуем использования методов типа setMinimumNumAlphaBits(). По-умолчанию все параметры равны 0.
4. Рзарешение использования сглаживание и его глубина, при помощи метода setNumMultiSamples(). По-умолчаниею - 0.
5. Включение стерео-режима. Выключен по-умолчанию.

Рассмотрим использование данного синглтона на примере сглаживания

**main.h**
```cpp
#ifndef		MAIN_H
#define		MAIN_H

#include    <osgDB/ReadFile>
#include    <osgViewer/Viewer>

#endif
```

**main.cpp**
```cpp
#include	"main.h"

int main(int argc, char *argv[])
{
    (void) argc; (void) argv;

    osg::DisplaySettings::instance()->setNumMultiSamples(6);

    osg::ref_ptr<osg::Node> model = osgDB::readNodeFile("../data/cessna.osg");

    osgViewer::Viewer viewer;
    viewer.setSceneData(model.get());

    return viewer.run();
}
```

Существенным здесб является лишь один вызов

```cpp
osg::DisplaySettings::instance()->setNumMultiSamples(6);
```

- задание параметра сглаживания, который может принимать занчения 2, 4 и 6 в зависимости от используемого графического устройства. Обратим винмание, как выглядит лопасть винта цессны без применения сглаживания 

![](https://habrastorage.org/webt/4y/nr/1p/4ynr1p8ia9knpagauohb6_fe2-g.png)

и после его применения

![](https://habrastorage.org/webt/yq/4b/og/yq4bog1qdmngwxpsij7ujponxzk.png)