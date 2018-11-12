---
layout: post
title:  "Введение в OpenSceneGraph: Анимация состояния рендеринга"
date:   2018-11-12 20:07:00 +0300
categories: jekyll update
---

Состояния рендеринга также можно анимировать. Целый рад визуальных эффектов может быть сгенерирован путем изменения свойств одного или нескольких атрибутов рендеринга. Подбную анимацию, изменяющую состояние атрибутов рендеринга легко реализовать через механизм обратных вызовов при обновлении сцены.

Классы стандартных интерполяций так же мотут быть использованы для задания функции изменения параметров атрибутов.

У нас уже имеется опыт создания полупрозрачных объектов. Мы знаем, что если альфа-компонента цвета равна нулю, мы получаем полностью прозрачный объект, при значении 1 - полностью непрозрачный. Ясно, что варьируя этот параметр от 0 до 1 во времени можно получить эффект постепенного появления или исчезания объекта. Проиллюстрируем это на конкретном примере

**main.h**
```cpp
#ifndef		MAIN_H
#define		MAIN_H

#include    <osg/Geode>
#include    <osg/Geometry>
#include    <osg/BlendFunc>
#include    <osg/Material>
#include    <osgAnimation/EaseMotion>
#include    <osgDB/ReadFile>
#include    <osgViewer/Viewer>

#endif
```

**main.cpp**
```cpp
#include	"main.h"

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
class AlphaFadingCallback : public osg::StateAttributeCallback
{
public:

    AlphaFadingCallback()
    {
        _motion = new osgAnimation::InOutCubicMotion(0.0f, 1.0f);
    }

    virtual void operator() (osg::StateAttribute* , osg::NodeVisitor*);

protected:

    osg::ref_ptr<osgAnimation::InOutCubicMotion> _motion;
};

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
void AlphaFadingCallback::operator()(osg::StateAttribute *sa, osg::NodeVisitor *nv)
{
    (void) nv;

    osg::Material *material = static_cast<osg::Material *>(sa);

    if (material)
    {
        _motion->update(0.0005f);

        float alpha = _motion->getValue();

        material->setDiffuse(osg::Material::FRONT_AND_BACK, osg::Vec4(0.0f, 1.0f, 1.0f, alpha));
    }
}

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
int main(int argc, char *argv[])
{
    (void) argc; (void) argv;

    osg::ref_ptr<osg::Drawable> quad = osg::createTexturedQuadGeometry(
                osg::Vec3(-0.5f, 0.0f, -0.5f),
                osg::Vec3(1.0f, 0.0f, 0.0f),
                osg::Vec3(0.0f, 0.0f, 1.0f));

    osg::ref_ptr<osg::Geode> geode = new osg::Geode;
    geode->addDrawable(quad.get());

    osg::ref_ptr<osg::Material> material = new osg::Material;
    material->setAmbient(osg::Material::FRONT_AND_BACK, osg::Vec4(0.0f, 0.0f, 0.0f, 1.0f));
    material->setDiffuse(osg::Material::FRONT_AND_BACK, osg::Vec4(0.0f, 1.0f, 1.0f, 0.5f));
    material->setUpdateCallback(new AlphaFadingCallback);

    geode->getOrCreateStateSet()->setAttributeAndModes(material.get());
    geode->getOrCreateStateSet()->setAttributeAndModes(new osg::BlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA));
    geode->getOrCreateStateSet()->setRenderingHint(osg::StateSet::TRANSPARENT_BIN);

    osg::ref_ptr<osg::Group> root = new osg::Group;
    root->addChild(geode.get());
    root->addChild(osgDB::readNodeFile("../data/glider.osg"));

    osgViewer::Viewer viewer;
    viewer.setSceneData(root.get());
    
    return viewer.run();
}
```

Начинаем с создания обратного вызова-обработчика измнения значения альфа-канала во времени

```cpp
class AlphaFadingCallback : public osg::StateAttributeCallback
{
public:

    AlphaFadingCallback()
    {
        _motion = new osgAnimation::InOutCubicMotion(0.0f, 1.0f);
    }

    virtual void operator() (osg::StateAttribute* , osg::NodeVisitor*);

protected:

    osg::ref_ptr<osgAnimation::InOutCubicMotion> _motion;
};
```

Защишенный параметр _motion будет определять ту функцию, по которой будет изменятся значение альфы во времени. Для данного примера выберем аппроксимацию кубическим сплайном, задавая её сразу же, в конструкторе класса

```cpp
AlphaFadingCallback()
{
    _motion = new osgAnimation::InOutCubicMotion(0.0f, 1.0f);
}
```

Эта зависимость может быть проиллюстрирована вот такой кривой

![](https://habrastorage.org/webt/ej/gd/cr/ejgdcr97gkb0lru6evved46sk6q.png)

В кострукторе объекта InOutCubicMotion определяем пределы изменения аппроксимируемой величины от 0 до 1. Далее переопределяем operator() для данного класса таким образом

```cpp
void AlphaFadingCallback::operator()(osg::StateAttribute *sa, osg::NodeVisitor *nv)
{
    (void) nv;

    osg::Material *material = static_cast<osg::Material *>(sa);

    if (material)
    {
        _motion->update(0.0005f);

        float alpha = _motion->getValue();

        material->setDiffuse(osg::Material::FRONT_AND_BACK, osg::Vec4(0.0f, 1.0f, 1.0f, alpha));
    }
}
```

Получаем указатель на материал

```cpp
osg::Material *material = static_cast<osg::Material *>(sa);
```

В коллбэк приходит абстрактрое значение аттрибута, однако мы прикрепим данный обработчик к материалу, поэтому придет  имеенно указатель на материал, поэтому мы смело преобразуем атрибут состояния к указателю на материал. Далее мы задаем интервал времени обновления аппроксимирующей функции - чем он больше, тем быстрее будет происходить изменение параметра в пределах заданного диапазона

```cpp
_motion->update(0.0005f);
```

Читаем значение аппроксимирующей функции

```cpp
float alpha = _motion->getValue();
```

и присваиваем материалу новое значение диффузного цвета

```cpp
material->setDiffuse(osg::Material::FRONT_AND_BACK, osg::Vec4(0.0f, 1.0f, 1.0f, alpha));
```

Теперь сформируем сцену в функции main(). Я думаю вы уже устали каждый раз строить квадрат по вершинам, поэтому упростим задачу - сгенерируем квадратный полигон стандартной функцией OSG

```cpp
osg::ref_ptr<osg::Drawable> quad = osg::createTexturedQuadGeometry(
                osg::Vec3(-0.5f, 0.0f, -0.5f),
                osg::Vec3(1.0f, 0.0f, 0.0f),
                osg::Vec3(0.0f, 0.0f, 1.0f));
```

В качестве первого параметра выступает точка, от которой будет строится левый нижний угол квадрата, два других параметра задают координаты диагоналей. Разобравшись с квадратом, создаем для него материал

```cpp
osg::ref_ptr<osg::Material> material = new osg::Material;
material->setAmbient(osg::Material::FRONT_AND_BACK, osg::Vec4(0.0f, 0.0f, 0.0f, 1.0f));
material->setDiffuse(osg::Material::FRONT_AND_BACK, osg::Vec4(0.0f, 1.0f, 1.0f, 0.5f));
```

Мы указываем параметры цвета материала. Ambient color - это параметр, характеризующий цвет материала в затененной области, недоступной для источников цвета. Diffuse color - собственный цвет материала, характеризующий способность поверхности рассеивать падающий на неё цвет, то есть то, что мы привыкли называть цветом в быту. Параметр FRONT_AND_BACK указывает, что данный атрибут цвета присваивается как лицевой, так и обратной стороне граней геометрии. 

Назначаем материалу созданный нами ранее обработчик

```cpp
material->setUpdateCallback(new AlphaFadingCallback);
```

Назначаем созданный материал квадрату

```cpp
geode->getOrCreateStateSet()->setAttributeAndModes(material.get());
```

и задаем другие атрибуты - функцию смешивания цветов и указываем, что данный объект имеет прозрачные грани

```cpp
geode->getOrCreateStateSet()->setAttributeAndModes(new osg::BlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA));
geode->getOrCreateStateSet()->setRenderingHint(osg::StateSet::TRANSPARENT_BIN);
```

Завершаем формирование сцены и запускаем вьювер

```cpp
osg::ref_ptr<osg::Group> root = new osg::Group;
root->addChild(geode.get());
root->addChild(osgDB::readNodeFile("../data/glider.osg"));

osgViewer::Viewer viewer;
viewer.setSceneData(root.get());

return viewer.run();
```

Получаем результат в виде плавно появляющегося в сцене квадрата

![](https://habrastorage.org/webt/s5/jv/1h/s5jv1h_ebaecadz6owo60nnrdvg.gif)


## Небольшая ремарка о зависимостях

Наверняка ваш пример не компилируется, выдавая ошибку на этапе компоновки. Это не случайно -- обратите внимание на строчку в заголовочном файле main.h

```cpp
#include    <osgAnimation/EaseMotion>
```

Каталог заголовков OSG, из которого берется заголовочный файл, обычно указывает на ту библиотеку, в которой содержится реализация функций и классов, описанных в заголовке. Пожтому появление каталога osgAnimation/ должно наводить на мысль о том, что в список линковки сценврия сборки проекта следует добавить одноименную бибилиотеку, примерно так (с учетом путей к бибилиотеким и версии сборки)

```cmake
LIBS += -losgAnimation
```

