---
layout: post
title:  "Введение в OpenSceneGraph: Основы работы с текстурами"
date:   2018-11-06 16:02:00 +0300
categories: jekyll update
---

Мы уже рассматривали пример, где раскрашивали квадрат во все цвета радуги. Тем не менее существует и другая технология, а именно применение к трехмерной геометрии так называемой *текстурной карты* или просто текстуры - растрового двухмерного изображения. При этом воздействие оказывается не на вершины геометрии, а изменяются данные всех пикселей, получаемых при растеризации сцены. Такой прием позволяет существенно увеличить реалистичность и детальность конечного изображения.

OSG поддерживает несколько текстурных атрибутов и режимов текстурирования. Но, перед тем как говорить о текстурах, поговорим о том, как в OSG оперируют с растровыми изображениями. Для работы с растровыми изображениями предусмотрен специальный класс - osg::Image, хранящий внутри себя данные изображения, предназначенных, в конечном итоге, для текстурирования объекта.

## Представление данных растровых изображений. Класс osg::Image

Лучшим способом загрузки изображения с диска служит применение вызова osgDB::readImageFile(). Оно очень похож на уже набившbq нам оскомину вызов osg::readNodeFile(). Если у нас есть битмэп с именем picture.bmp, то есго загрузка будет выглядеть так

```cpp
osg::ref_ptr<osg::Image> image = osgDB::readImageFile("picture.bmp");
```

Если изображение загружено корректно, то указатель будет валидным, в противном случае функция возвратит NULL. После загрузки мы можем получить информацию об изображении, используя следующие публичные методы

1. s(), t() и r() - вовращают ширину, высоту и глубину изображения.
2. data() - возвращает указатель типа unsigned char* на "сырые" данные изображения. Через данный указатель разработчик может непосредственно воздействовать на данные изображения. Получить представление о формате данных изображения можно, используя методы getPixalFormat() и getDataType(). Возващаемые ими значения эквивалентны параметрам формата и типа функций OpenGL glTexImage*(). Например, если картинка имеет формат пикселя GL_RGB и тип данный GL_UNSIGNED_BYTE то используются три независимых элемена (беззнаковых байта) для представления RGB-компонент цвета

![](https://habrastorage.org/webt/6g/mo/or/6gmoorwfvryg44pq7sdi0cb4l9s.png)

Можно создать новый объект изображения и выделить под него память

```cpp
osg::ref_ptr<osg::Image> image = new osg::Image;
image->allocateImage(s, t, r, GL_RGB, GL_UNSIGNED_BYTE);
unsigned char *ptr = image->data();
// Далее выполняем с буфером данных изображения любые операции 
```

Здесь s, t, r - размеры изображения; GL_RGB задает формат пикселя, а GL_UNSIGNED_BYTE - тип данных для описания одной цветовой компоненты. Внетренний буфер данных нужного размера выделяется в памяти и автоматически уничтожается, если на данное изображение нет ни одной ссылки.

Система плагинов OSG поддерживает загрузку чуть ли не всех популярных форматов изобаржений: *.jpg, *.bmp, *.png, *.tif и так далее. Этот список нетрудно расширить, написав собственный плагин, но это тема для отдельной беседы.

## Основы текстурирования

Для наложения текстуры на трехмерную модель необходимо выполнить ряд шагов:

1. Задать геометрическому объекту текстурные координаты вершин (в среде трехмерных дизайнеров это называется UV-разверткой).
2. Создать объект атрибута текстуры для 1D, 2D, 3D или  кубической текстуры.
3. Задать одно или несколько изображения для атрибута текстуры.
4. Прикрепить текстурный атрибут и режим к набору состояний, прменяемому к отрисовываемому объекту.

OSG определяет класс osg::Texture, инкапсулирующий все виды текстур. От него производятся подклассы osg::Texture1D, osg::Texture2D, osg::Texture3D и osg::TextureCubeMap, которые представляю различные техники текстурирования, принятые в OpenGL.

Наиболее употребимый метод класса osg::Texture это setImage(), задающий изображение, используемое в текстуре, например

```cpp
osg::ref_ptr<osg::Image> image = osgDB::readImageFile("picture.bmp");
osg::ref_ptr<osg::Texture2D> texture = new osg::Texture2D;
texture->setImage(image.get());
```

или, можно передать объект изображения непосредственно в конструктор класса текстуры

```cpp
osg::ref_ptr<osg::Image> image = osgDB::readImageFile("picture.bmp");
osg::ref_ptr<osg::Texture2D> texture = new osg::Texture2D(image.get());
```

Изображение можно получить обратно из объекта текстуры вызвав метод getImage().

Другим важным моментом является задание текстурных координат для каждой вершины в объекта osg::Geometry. Передача этих координат происходит через массив osg::Vec2Array и osg::Vec3Array вызовом метода setTexCoordArray().

После задания текстурных координат мы должны установить номер текстурного слота (юнит), так как OSG поддерживает наложение нескольких текстур на одну и ту же геометрию. При спользовании одной текстуры номер юнита всегда равен 0ю Например, следующий код иллюстрирует задание текстурных координат для юнита 0 геометрии

```cpp
osf::ref_ptr<osg::Vec2Array> texcoord = new osg::Vec2Array;
texcoord->push_back( osg::Vec2(...) );
...
geom->setTexCoordArray(0, texcoord.get());
```

После этого мы можем добавить атрибут текстуры в набор состояний, автоматически включая соответствующий режим текстурирование (в нашем примере GL_TEXTURE_2D) и применить атрибут к геометрии или узлу, содержащему данную геометрию

```cpp
geom->getOrCreateStateSet()->setTextureAttributeAndModes(texture.get());
```

Обращаем внимание на то, что OpenGL управляет данными изображения в графической памяти видеокарты, но объект osg::Image вместе с теми же данными располагается в системной памяти. В резултате мы столкнемся с тем, что у нас хранятся два экземпляра одних и тех же данных, занимая память процесса. Если данное изображение не используется совместно несколькими атрибутами текстуры, его можно удалить из системной памяти сразу после того как OpenGL перенесет из в память видеоадаптера. Для включения этой функции класс osg::Texture предоставляет соотвествующий метод

```cpp
texture->setUnRefImageDataAfterApply( true );
```

## Загружаем и применяем 2D-текстуру

**main.h**
```cpp
#ifndef		MAIN_H
#define		MAIN_H

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

    osg::ref_ptr<osg::Geometry> quad = new osg::Geometry;
    quad->setVertexArray(vertices.get());
    quad->setNormalArray(normals.get());
    quad->setNormalBinding(osg::Geometry::BIND_OVERALL);
    quad->setTexCoordArray(0, texcoords.get());
    quad->addPrimitiveSet( new osg::DrawArrays(GL_QUADS, 0, 4) );

    osg::ref_ptr<osg::Texture2D> texture = new osg::Texture2D;
    osg::ref_ptr<osg::Image> image = osgDB::readImageFile("../data/Images/lz.rgb");
    texture->setImage(image.get());

    osg::ref_ptr<osg::Geode> root = new osg::Geode;
    root->addDrawable(quad.get());
    root->getOrCreateStateSet()->setTextureAttributeAndModes(0, texture.get());

    osgViewer::Viewer viewer;
    viewer.setSceneData(root.get());
    
    return viewer.run();
}
```

![](https://habrastorage.org/webt/wm/2a/fy/wm2afywfx6dnkhwaagvceqkycd8.png)