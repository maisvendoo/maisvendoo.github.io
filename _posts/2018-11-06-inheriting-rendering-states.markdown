---
layout: post
title:  "Введение в OpenSceneGraph: Наследование состояний рендеринга. Применение атрибутов и режимов"
date:   2018-11-06 10:05:00 +0300
categories: jekyll update
---

Набор состояний узла влияет на текущий узел и все его дочерние элементы. Например, атрибут osg::PolygonMode, установленный для узла transform1 из предыдущего примера будет применен ко всем дочерним элементам этого узла. Однако, дочерний узел может переопределять родительские атрибуты, то есть состояние рендеринга будет наследоваться от родительского узла, если дочерний узел не изменит поведение.

![](https://habrastorage.org/webt/ez/_c/4q/ez_c4qowpsmgjqwqmhtpu5noilm.png)

Иногда тербуется переопределить поведение узла в части применения аттрибутов. Например, в большинстве 3D-редакторов пользователь может загрузить несколько моделей и менять режим их отображения для всех загруженных моделей одновременно, независимо от того, каким образом они отображались ранее. Другими словами, все модели в редакторе должны наследовать единый атрибут независимо от того, как его задавали раньше для кажой из моделей. В OSG это может быть реализовано с помощью флага osg::StateAttribute::OVERRIDE, например

```cpp
stateset->StateAttribute(attr, osg::StateAttribute::OVERRIDE);
```

При установке режимов и режимов с атрибутами используются оператор побитового ИЛИ

```cpp
stateset->StateAttributeAndModes(attr, osg::StateAttribute::ON | osg::StateAttribute::OVERRIDE);
```

Кроме того, возможна и защита атрибута от переопределения - для этого он должен быть помечен флагом osg::StateAttribute::PROTECTED.

Имеется и третий флаг, osg::StateAttribute::INHERIT, который используется для того, чтобы отметить, что данный атрибут должен наследоваться из набора состояний родительского узла.

Приведем короткий пример использвание флагов OVERRIDE и PROTECTED. Корневой узел будет установлен в OVERRIDE, чтобы заставить все дочерние узлы наследовать его атрибуты и режимы. При этом дочерние узлы будут пытаться изменить свое состояние с помощью или без помощи флага PROTECTED, что будет приводить к различным результатам.

**main.h**
```cpp
#ifndef		MAIN_H
#define		MAIN_H

#include    <osg/PolygonMode>
#include    <osg/MatrixTransform>
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

    osg::ref_ptr<osg::Node> model = osgDB::readNodeFile("../data/glider.osg");

    osg::ref_ptr<osg::MatrixTransform> transform1 = new osg::MatrixTransform;
    transform1->setMatrix(osg::Matrix::translate(-0.5f, 0.0f, 0.0f));
    transform1->addChild(model.get());

    osg::ref_ptr<osg::MatrixTransform> transform2 = new osg::MatrixTransform;
    transform2->setMatrix(osg::Matrix::translate(0.5f, 0.0f, 0.0f));
    transform2->addChild(model.get());

    osg::ref_ptr<osg::Group> root = new osg::Group;
    root->addChild(transform1.get());
    root->addChild(transform2.get());

    transform1->getOrCreateStateSet()->setMode(GL_LIGHTING, osg::StateAttribute::OFF);
    transform2->getOrCreateStateSet()->setMode(GL_LIGHTING, osg::StateAttribute::OFF | osg::StateAttribute::PROTECTED);
    root->getOrCreateStateSet()->setMode(GL_LIGHTING, osg::StateAttribute::ON | osg::StateAttribute::OVERRIDE);

    osgViewer::Viewer viewer;
    viewer.setSceneData(root.get());
    
    return viewer.run();
}
```

![](https://habrastorage.org/webt/lf/8h/u4/lf8hu4n3ii1bor6lgsktbischd0.png)

Чтобы понять, что вообще происходит, необходимо посмотреть как выглядит нормально освещенный дельтаплан, загрузив его  штатный просмотрищик OSG osgviewer

```bash
$ osgviewer glider.osg
```

В примере мы пытаемся поменять режим освещения для узлов transform1 и transform2, отключив напрочь освещение. 

```cpp
transform1->getOrCreateStateSet()->setMode(GL_LIGHTING, osg::StateAttribute::OFF);
transform2->getOrCreateStateSet()->setMode(GL_LIGHTING, osg::StateAttribute::OFF | osg::StateAttribute::PROTECTED);
```

При этом мы включаем режим освещения для корневого узла, и, используя флаг OVERRIDE для всех его дочерних узлов, чтобы они наследовали состояние корневого узла. Однако trnsform2 использует флаг PROTECTED для предотварщения вилияния настроек корневого узла. 

```cpp
transform2->getOrCreateStateSet()->setMode(GL_LIGHTING, osg::StateAttribute::OFF | osg::StateAttribute::PROTECTED);
```

В итоге, несмотря на то, что мы выключаем освещение у узла transform1 левый дельтаплан по-прежнему освещен, так как настройки корня сцены перекрыли нашу попытку выключить освещение для него. Правый дельтаплан отображается без освещения (он выглядит ярче только потому, что залит простым цветом без просчета освещенности), так как transform2 защищен от наследования аттрибутов корневого узла.

## Перечень атрибутов OpenGL, поддерживаемых в OpenSceneGraph

OSG поддерживает почти все атрибуты и режимы рендеринга, поддерживаемые OpenGL, через классы, производные от osg::StateAttribute. В таблице представлены все параметры машины состояния OpenGL, доступные из движка.

|----------------|--------------------|----------------------------|------------------------------------------------|
|ID типа атрибута|Имя класса          |Ассоциированный режим       |Эквивалентная функция OpenGL                    |
|----------------|--------------------|----------------------------|------------------------------------------------|
|ALPHEFUNC       |osg::AlphaFunc      |GL_ALPHA_TEST               |glAlphaFunc()                                   |
|BLENDFUNC       |osg::BlendFunc      |GL_BLEND                    |glBlendFunc() и glBlendFuncSeparate()           |
|CLIPPLANE       |osg::ClipPlane      |GL_CLIP_PLANEi (i от 1 до 5)|glClipPlane()                                   |
|COLORMASK       |osg::ColorMask      |--                          |glColorMask()                                   |
|CULLFACE        |osg::CullFace       |GL_CULLFACE                 |glCullFace()                                    |
|DEPTH           |osg::Depth          |GL_DEPTH_TEST               |glDepthFunc(), glDepthRange() и glDepthMask()   |
|FOG             |osg::Fog            |GL_FOG                      |glFog()                                         |
|FRONTFACE       |osg::FrontFace      |--                          |glFrontFace()                                   |
|LIGHT           |osg::Light          |GL_LIGHTi (i от 1 до 7)     |glLight()                                       |
|LIGHTMODEL      |osg::LightModel     |--                          |glLightModel()                                  |
|LINESTRIPPLE    |osg::LineStripple   |GL_LINE_STRIPPLE            |glLineStripple()                                |
|LINEWIDTH       |osg::LineWidth      |--                          |glLineWidht()                                   |
|LOGICOP         |osg::LogicOp        |GL_COLOR_LOGIC_OP           |glLogicOp()                                     |
|MATERIAL        |osg::Material       |--                          |glMaterial() и glColorMaterial()                |
|POINT           |osg::Point          |GL_POINT_SMOOTH             |glPointParameter()                              |
|POINTSPRITE     |osg::PointSprite    |GL_POINT_SPRITE_ARB         |Функции для работы со спрайтами OpenGL          |
|POLYGONMODE     |osg::PolygonMode    |--                          |glPolygonMode()                                 |
|POLYGONOFFSET   |osg::PolygonOffset  |GL_POLYGON_OFFSET_POINT     |glPolygonOffset()                               |
|POLYGONSTRIPPLE |osg::PolygonStripple|GL_POLYGON_STRIPPLE         |glPolygonStripple()                             |
|SCISSOR         |osg::Scissor        |GL_SCISSOR_TEST             |glScissor()                                     |
|SHADEMODEL      |osg::ShadeModel     |--                          |glShadeModel()                                  |
|STENCIL         |osg::Stencil        |GL_STENCIL_TEST             |glStencilFunc(), glStencilOp() и glStencilMask()|
|TEXENV          |osg::TexEnv         |--                          |glTexEnv()                                      |
|TEXGEN          |osg::TexGen         |GL_TEXTURE_GEN_S            |glTexGen()                                      |

Колонка ID типа атрибута указывает на специфический идентификатор OSG, которым обозначается данный атрибут в перечислителях класса osg::StateAttribute. Он может быть использован в методе getAttribute, для получения значения конкретного атрибута

```cpp
osg::PolygonMode *pm = dynamic_cast<osg::PolygonMode *>(stateset->getAttribute(osg::StateAttribute::POLYGONMODE));
```

Валидный указатель указывает на то, что атрибут был установлен ранее. В противном случае метод вренет NULL. Получить значение текущего режима можно также, использовав вызов

```cpp
osg::StateAttribute::GLModeValue value = stateset->getMode(GL_LIGHTING);
```

Здесь перечислитель GL_LIGHTING используется для включения/отключения освещения на всей сцене.

## Применение тумана к модели в сцене

Приведем в пример эффект тумана, как идеальный способ показать приемы работы с различными атрибутами и режимами рендеринга. OpenGL использует одно линейное и два экспоненциальных уравнения, описывающих модель тумана, поддерживаемые классом osg::Fog.

**main.h**
```cpp
#ifndef		MAIN_H
#define		MAIN_H

#include    <osg/Fog>
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

    osg::ref_ptr<osg::Fog> fog = new osg::Fog;
    fog->setMode(osg::Fog::LINEAR);
    fog->setStart(500.0f);
    fog->setEnd(2500.0f);
    fog->setColor(osg::Vec4(1.0f, 1.0f, 0.0f, 1.0f));

    osg::ref_ptr<osg::Node> model = osgDB::readNodeFile("../data/lz.osg");
    model->getOrCreateStateSet()->setAttributeAndModes(fog.get());
    
    osgViewer::Viewer viewer;
    viewer.setSceneData(model.get());

    return viewer.run();
}
```

Первым делом создаем атрибут тумана. Используем линейную модель, настраиваем диапазон отображения тумана по дальности до модели

```cpp
osg::ref_ptr<osg::Fog> fog = new osg::Fog;
fog->setMode(osg::Fog::LINEAR);
fog->setStart(500.0f);
fog->setEnd(2500.0f);
fog->setColor(osg::Vec4(1.0f, 1.0f, 0.0f, 1.0f));
```

Загружаем образец ландшафта lz.osg и применяем к нему данный атрибут

```cpp
osg::ref_ptr<osg::Node> model = osgDB::readNodeFile("../data/lz.osg");
model->getOrCreateStateSet()->setAttributeAndModes(fog.get());
```

В окне вьювера видим затуманенный ландшафт, и можем посмотреть как изменяется густота тумана в зависимости от расстояния до модели

![](https://habrastorage.org/webt/gq/8f/fu/gq8ffuswpsbsl1-xxqzprwvprle.png)

![](https://habrastorage.org/webt/3z/st/f6/3zstf6mlxlvenapqhvj3zsxyh2i.png)

![](https://habrastorage.org/webt/k-/mc/cj/k-mccjrssmakzwwrht-b5azkzg4.png)