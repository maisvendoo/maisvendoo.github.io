---
layout: post
title:  "Введение в OpenSceneGraph: Работа с источниками света и оcвещением"
date:   2018-11-06 14:20:00 +0300
categories: jekyll update
---

Как и в OpenGL, OSG поддерживает до восьми источников света с фиксированной функцией для непосредственного совещения объектов сцены. Как и OpenGL, OSG не умеет автоматически расчитывать тени. Световые лучи идут от источников по прямым линиям, отражаются от объектов и рассеиваются ими, после чего воспринимаются глазами зрителя. Для качественно обработки освещения необходимо задавать свойства материала, нормали геометрии объектов и т.д.

Класс osg::Light предоставляет методы для управления источниками света, включающие: setLightNum() и getLightNum() - для работы с количеством источников; setAmbient() и getAmbient() для управления окружающей компонентой; setDiffuse() и getDiffuse() - для работы с рассеянной компонентой и т.д.

Кроме того, OSG описывает класс osg::LightSource для добавления источников света в сцену. Он предоставляет метод setLight() и является листовым узлом графа сцены с единственным атрибутом. Все остальные узлы графа сцены испытывают на себе влияние истояника света если установлен соотвествующие режим для GL_LIGHTi. Например:

```cpp
// Источник света 1
osg::ref_ptr<osg::Light> light = new osg::Light;
light->setLightNum( 1 ); 
...
// Содаем источник света как узлел сцены
osg::ref_ptr<osg::LightSource> lightSource = new osg::LightSource;
lightSource->setLight( light.get() ); 
...
// Доабвляем созданный узел в коренвой узел сцены и устанавливаем режим для него
root->addChild( lightSource.get() );
root->getOrCreateStateSet()->setMode( GL_LIGHT1,
osg::StateAttribute::ON );

```

Другим, более удобным решением является метод setStateSetModes(), с помощью которого источник света с нужным номером автоматически присоединяется к корневому узлу

```cpp
root->addChild( lightSource.get() );
lightSource->setStateSetModes( root->getOrCreateStateSet(), osg::StateAttribute::ON );
```

Можно добавить дочерние узлы к источнику света, но это совершенно не означает, вы будете освещать связанный с ним подграф как-то по особому. Он будет обрабатываться как геометрия, представленая физической формой источника света.

Узел osg::LightSource может быть прикреплен к узлу трансформации, и, например точесный источник света можно будет перемещать в пространстве. Это можно отключить, установив для источника света абсолютную систему координат

```cpp
lightSource->setReferenceFrame( osg::LightSource::ABSOLUTE_RF );
```

## Создание источников света в сцене

По-умолчанию OSG автоматически настраивает источник света с номером 0, излучающий на сцену равномерный направленный свет. Однако, в любой момент можно добавить несколько дополнительных источников света, да ещё и управлять ими, применяя узлы трансформации координат. Перемещению можно подвергать только позиционные источники (точечные). Направленный свет имеет только направление (поток параллельных лучей идущих из бесконечности) и не имеет привязки к конкретному положению на сцене. OpenGL и OSG импользуют четвертый компонент параметра положения чтобы задать тип источника света. Если он равен 0, то свет рассматривается как направленный; при значении равном 1 - позиционный.

Рассмотрим небольшой пример работы с освещением.

**main.h**
```cpp
#ifndef		MAIN_H
#define		MAIN_H

#include    <osg/MatrixTransform>
#include    <osg/LightSource>
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
osg::Node *createLightSource(int num,
                             const osg::Vec3 &trans,
                             const osg::Vec4 &color)
{
    osg::ref_ptr<osg::Light> light = new osg::Light;
    light->setLightNum(num);
    light->setDiffuse(color);
    light->setPosition(osg::Vec4(0.0f, 0.0f, 0.0f, 1.0f));

    osg::ref_ptr<osg::LightSource> lightSource = new osg::LightSource;
    lightSource->setLight(light);

    osg::ref_ptr<osg::MatrixTransform> sourceTrans = new osg::MatrixTransform;
    sourceTrans->setMatrix(osg::Matrix::translate(trans));
    sourceTrans->addChild(lightSource.get());

    return sourceTrans.release();
}

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
int main(int argc, char *argv[])
{
    (void) argc; (void) argv;

    osg::ref_ptr<osg::Node> model = osgDB::readNodeFile("../data/cessna.osg");

    osg::ref_ptr<osg::Group> root = new osg::Group;
    root->addChild(model.get());

    osg::Node *light0 = createLightSource(0, osg::Vec3(-20.0f, 0.0f, 0.0f),
                                          osg::Vec4(1.0f, 1.0f, 0.0f, 1.0f));

    osg::Node *light1 = createLightSource(1, osg::Vec3(0.0f, -20.0f, 0.0f),
                                          osg::Vec4(0.0f, 1.0f, 1.0f, 1.0f));

    root->getOrCreateStateSet()->setMode(GL_LIGHT0, osg::StateAttribute::ON);
    root->getOrCreateStateSet()->setMode(GL_LIGHT1, osg::StateAttribute::ON);

    root->addChild(light0);
    root->addChild(light1);

    osgViewer::Viewer viewer;
    viewer.setSceneData(root.get());
    
    return viewer.run();
}
```

Для создания источника света заведем отдельную функцию

```cpp
osg::Node *createLightSource(int num,
                             const osg::Vec3 &trans,
                             const osg::Vec4 &color)
{
    osg::ref_ptr<osg::Light> light = new osg::Light;
    light->setLightNum(num);
    light->setDiffuse(color);
    light->setPosition(osg::Vec4(0.0f, 0.0f, 0.0f, 1.0f));

    osg::ref_ptr<osg::LightSource> lightSource = new osg::LightSource;
    lightSource->setLight(light);

    osg::ref_ptr<osg::MatrixTransform> sourceTrans = new osg::MatrixTransform;
    sourceTrans->setMatrix(osg::Matrix::translate(trans));
    sourceTrans->addChild(lightSource.get());

    return sourceTrans.release();
}
```

В этой функции мы сперва определяем параметры освещения, даваемые источником, создавая тем самым атрибут GL_LIGHTi

```cpp
osg::ref_ptr<osg::Light> light = new osg::Light;
// Номер источника света
light->setLightNum(num);
// Цвет источника
light->setDiffuse(color);
// Положение источника. Последний компонент указывает на то, что источник точечный
light->setPosition(osg::Vec4(0.0f, 0.0f, 0.0f, 1.0f));
```

После этого создается источник света, которому присваивается данный атрибут

```cpp
osg::ref_ptr<osg::LightSource> lightSource = new osg::LightSource;
lightSource->setLight(light);
```

Создаем и настраиваем узел трансформации, передавая ему наш источник света в качестве дочернего узла

```cpp
osg::ref_ptr<osg::MatrixTransform> sourceTrans = new osg::MatrixTransform;
sourceTrans->setMatrix(osg::Matrix::translate(trans));
sourceTrans->addChild(lightSource.get());
```

Возвращаем указатель на узел трансформации

```cpp
return sourceTrans.release();
```

В теле основной программы мы загружаем трехмерную модель (снова наша любимая цессна)

```cpp
osg::ref_ptr<osg::Node> model = osgDB::readNodeFile("../data/cessna.osg");

osg::ref_ptr<osg::Group> root = new osg::Group;
root->addChild(model.get());
```

Создаем два источника света с номерами 0 и 1. Первый будет светить желтым светом, второй - сине-зеленым

```cpp
osg::Node *light0 = createLightSource(0, osg::Vec3(-20.0f, 0.0f, 0.0f),
                                          osg::Vec4(1.0f, 1.0f, 0.0f, 1.0f));

osg::Node *light1 = createLightSource(1, osg::Vec3(0.0f, -20.0f, 0.0f),
                                      osg::Vec4(0.0f, 1.0f, 1.0f, 1.0f));
```

Сообщаем машине состояний OpenGL что необходимо включить 0 и 1 источники света и добавяем созданные нами источники в сцену

```cpp
root->getOrCreateStateSet()->setMode(GL_LIGHT0, osg::StateAttribute::ON);
root->getOrCreateStateSet()->setMode(GL_LIGHT1, osg::StateAttribute::ON);

root->addChild(light0);
root->addChild(light1);
```

После инициализации и запуска вьювера получаем картинку

![](https://habrastorage.org/webt/0s/or/ph/0sorphad56jc2zc8iwfb591qcxk.png)

