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

![](https://habrastorage.org/webt/s5/jv/1h/s5jv1h_ebaecadz6owo60nnrdvg.gif)