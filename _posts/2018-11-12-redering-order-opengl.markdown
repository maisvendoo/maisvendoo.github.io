---
layout: post
title:  "Введение в OpenSceneGraph: Порядок рендеринга примитивов в OpenGL"
date:   2018-11-12 16:49:00 +0300
categories: jekyll update
---

OpenGL хранит данные вершин и примитивов в различных буферах, таких как буфер цвета (color buffer), буфер глубины (depth buffer), буфер трафарета (stencil buffer) и так далее. Кроме того, он не перезаписывает вершины и треугольные грани уже отправленные в его конвейер. Это означает, что OpenGL создает новую геометрию вне зависимости от того, каким образом создавалась уже существующая геометрия. Это означает, что порядок в котором примитивы послылаются в конвейер рендеринга существенно влияет на конечный результат, который мы видим на экране.

Опираясь на данные буфера глубины OpenGL правильно отрисует непрозрачные объекты, сортируя пиксели по степени их удаленности от наблюдателя. Тем не менее, при использования техники смешивания цветов, например при реализации прозрачных и полупрозрачных объектов, будет выполнятся специальная операция обновления буфера цвета. Новые и старые пиксели изображения смешаваются, с учетом значения альфа-канала (четвертый компонент цвета). Это приводит к тому, что порядок рендеринга полурозрачных (translucent)  и непрозрачных (opaque) граней влияет на конечный результат

![](https://habrastorage.org/webt/tz/3w/o4/tz3wo4dfy14oscogqvrb1akmh5m.png)

на рисунке, в ситуации слева в конвейер были отправлены сначала непрозрачные, а потом прозрачные объекты, что привело к корректному смещению в буфере цвета и корректному отображению граней. В правой ситуации сначала были отрисованы прозрачные объекты, а затем непрозрачные, что привело к неверному отображению.

Метод setRenderingHint() класса osg::StateSet указывает OSG требуемый порядок рендеринга узлов и геометрических объектов, если это необходимо выполнить явно. Этот метод просто указывает, следует или не следует учитывать полупрозрачные грани при рендеринге, тем самым гарантируя, что в случае наличия в сцене полупрозрачных граней сначала будут отрисованы непрозрачные, а затем прозрачные грани, с учетом удаленности граней от наблюдателя. Чтобы сообщать движку, что данный узел непрозраный используем такой код

```cpp
node->getOrCreateStateSet()->setRenderingHint(osg::StateSet::OPAQUE_BIN);
```

или содержит прозрачные грани

```cpp
node->getOrCreateStateSet()->setRenderingHint(osg::StateSet::TRANSPARENT_BIN);
```

## Пример реализации полупрозрачных объектов
 
Попробуем проиллюстрировать всё вышеописанное теоретическое введение конкретным примером реализации полупрозрачного объекта.


**main.h**
```cpp
#ifndef		MAIN_H
#define		MAIN_H

#include    <osg/BlendFunc>
#include    <osg/Texture2D>
#include    <osg/Geometry>
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

    osg::ref_ptr<osg::Vec3Array> vertices = new osg::Vec3Array;
    vertices->push_back( osg::Vec3(-0.5f, 0.0f, -0.5f) );
    vertices->push_back( osg::Vec3( 0.5f, 0.0f, -0.5f) );
    vertices->push_back( osg::Vec3( 0.5f, 0.0f,  0.5f) );
    vertices->push_back( osg::Vec3(-0.5f, 0.0f,  0.5f) );

    osg::ref_ptr<osg::Vec3Array> normals = new osg::Vec3Array;
    normals->push_back( osg::Vec3(0.0f, -1.0f, 0.0f) );

    osg::ref_ptr<osg::Vec2Array> texcoords = new osg::Vec2Array;
    texcoords->push_back( osg::Vec2(0.0f, 0.0f) );
    texcoords->push_back( osg::Vec2(0.0f, 1.0f) );
    texcoords->push_back( osg::Vec2(1.0f, 1.0f) );
    texcoords->push_back( osg::Vec2(1.0f, 0.0f) );

    osg::ref_ptr<osg::Vec4Array> colors = new osg::Vec4Array;
    colors->push_back( osg::Vec4(1.0f, 1.0f, 1.0f, 0.5f) );

    osg::ref_ptr<osg::Geometry> quad = new osg::Geometry;
    quad->setVertexArray(vertices.get());
    quad->setNormalArray(normals.get());
    quad->setNormalBinding(osg::Geometry::BIND_OVERALL);
    quad->setColorArray(colors.get());
    quad->setColorBinding(osg::Geometry::BIND_OVERALL);
    quad->setTexCoordArray(0, texcoords.get());
    quad->addPrimitiveSet(new osg::DrawArrays(GL_QUADS, 0, 4));

    osg::ref_ptr<osg::Geode> geode = new osg::Geode;
    geode->addDrawable(quad.get());

    osg::ref_ptr<osg::Texture2D> texture = new osg::Texture2D;
    osg::ref_ptr<osg::Image> image = osgDB::readImageFile("../data/Images/lz.rgb");
    texture->setImage(image.get());

    osg::ref_ptr<osg::BlendFunc> blendFunc = new osg::BlendFunc;
    blendFunc->setFunction(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);

    osg::StateSet *stateset = geode->getOrCreateStateSet();
    stateset->setTextureAttributeAndModes(0, texture.get());
    stateset->setAttributeAndModes(blendFunc);    

    osg::ref_ptr<osg::Group> root = new osg::Group;
    root->addChild(geode.get());
    root->addChild(osgDB::readNodeFile("../data/glider.osg"));

    osgViewer::Viewer viewer;
    viewer.setSceneData(root.get());
    
    return viewer.run();
}
```

В большинстве своем приведенный здесь код не содержит ничего нового: создаются два геометрических объекта - текстурированный квадрат и дельтаплан, модель которого загружвается из файла. Однако, ко всем вершинам квадрата мы применяем белый полупрозрачный цвет

```cpp
colors->push_back( osg::Vec4(1.0f, 1.0f, 1.0f, 0.5f) );
```

-- значение альфа-канала равно 0.5, что в смешинии с цветами текстуры должно дать эффект полупрозрачного объекта. Кроме того, для обработки прозрачности следует задать функцию смешивания цветов

```cpp
osg::ref_ptr<osg::BlendFunc> blendFunc = new osg::BlendFunc;
blendFunc->setFunction(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);
```

передав её машине состояний OpenGL

```cpp
stateset->setAttributeAndModes(blendFunc);
```

При компиляции и запуске этой программы мы получим следующий результат

![](https://habrastorage.org/webt/_m/mz/kz/_mmzkzrmxvmfll-qi0xli6wq_5g.png)

Стоп! А где же прозрачность? Всё дело в том, что мы забыли указать движку, что следует обрабатывать прозрачные грани, что решается легко вызовом

```cpp
stateset->setRenderingHint(osg::StateSet::TRANSPARENT_BIN);
```

после чего мы получим нужный нам результат -- крыло дельтаплана просвечивается через полупрозрачный текстурированный квадрат

![](https://habrastorage.org/webt/py/bl/ta/pybltaqa6xkligouodxgunwoyvs.png)

Параметры функции смешивания GL_SRC_ALPHA и GL_ONE_MINUS_SRC_ALPHA означают, что результирующий пиксель экрана при отрисовке полупрозрачной грани будет иметь компоненты цвета, расчитываемые по формуле

```
R = srcR * srcA + dstR * (1 - srcA)
G = srcG * srcA + dstG * (1 - srcA)
B = srcB * srcA + dstB * (1 - srcA)
```

где [srcR, srcG, srcB] - компоненты цвета текстуры квадрата; [dstR, dstG, dstB] - компоненты цвета каждого пикселя того участка на который накладывается полупрозрачная грань, полученные с учетом того, что на этом месте уже отрисован фон и непрозрачные грани крыла дельтаплана. Под srcA понимаю альфа-компоненту цвета квадрата.

Метод seRenderingHint() отлчно упорядочивает отрисовку примитивов, но использовать его не слишком эффективно, так как сортировка прозрачных объектов по глубине при рендеринге кадра довольно ресурсоемкая операция. Поэтому разработчик должен позаботится о порядке отрисовки граней самостоятельно, если это возможно на предварительных этапах подготовки сцены.