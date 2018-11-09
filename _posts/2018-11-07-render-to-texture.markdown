---
layout: post
title:  "Введение в OpenSceneGraph: Рендеринг в текстуру"
date:   2018-11-07 15:20:00 +0300
categories: jekyll update
---

Техника рендеринга в текстуру позволяет разработчику создать текстуру, основанную на некоторой трехмерной подсцене или модели и применить её к поверхности на основной сцене. Подобную технологию часто называют "запеканием" текстуры.

Для динамического запекания текстуры необходимо выполнить три шага:

1. Создать объект текстуры для рендеринга в неё.
2. Отрендерить сцену в текстуру.
3. Использовать полученную текстуру по назначению.

Мы должны создать пустой текстурный объект. OSG позволяет сосздать пустую текстуру заданного размера. Метод setTextureSize() позволяет задавать ширину и высоту текстуры, а так же ещё глубину в качестве дополнительного параметра (для 3D-текстур).

Для выполнения рендеринга в текстуру её следует присоединить к объекту камеры путем вызова метода attach(), принимающего в качестве аргумента объект текстуры. Кроме того данный метод принимает аргрумент, указывающий, какую часть буфера кадра следует рендерить в данную текстуру. Например, для передачи буфера цвета в текстуру следует выполнить следующий код

```cpp
camera->attach( osg::Camera::COLOR_BUFFER, texture.get() );
```

К другим, доступным для рендеринга частям кадрового буфера, относятся буфер глубины DEPTH_BUFFER, буфер трафарета STENCIL_BUFFER дополнительные буферы цвета от COLOR_BUFFER0 до COLOR_BUFFER15. Наличие дополнительных буферов цвета и их количество определяется моделью видеокарты.

Кроме того, для камеры, выполняющей рендеринг в текстуру следует установить параметры матрицы проекции и вьюпорта, размер которого соотвествует размеру текстуры. Текстура будет обновлятся в процессе прорисовки каждого кадра. Необходимо учитывать, что основная камера не должна использоваться для рендеринга в текстуру, так как она обеспечивает рендеринг основной сцены и вы просто получите черный экран. Это требование может не выполнятся только тогда, когда вы выполняете внеэкранный рендеринг.

## Буфер кадра, пиксельный буфер и FBO

Проблема данной задачи в том, что необходим способ получения визуального представления буфера кадра и передача его в объект текстуры. Прямой подход к её решению заключается в использовании функций glReadPixels() для возврата пикселей из буферов кадра и применение результата в функции glTexImage* (). Это довольно простая операция, но она всегда будет выполнятся крайне медленно.

В плане повышения эффективности лучше использовать функцию glCopyTexSubImage(). Однако и этот способ оставляет ресурс для оптимизации. Хорошей идеей является рендеринг сцены напрямую в текстурый буфер изображения, не прибегая к копированию данных из буфера кадра. Для этого существует два основных решения:

1. Буфер для хранения данных о пикселях, с дескриптором формата пикселя, эквивалентный окну. Должен быть уничтоден по окончании рендеринга.
2. Объект буфера кадра (FBO), который иногда лучше пиксельного буфера в том смысле что позволяет добавлять буферы кадра и перенаправлять вывод на себя.

OSG поддерживает разные конечные точки рендеринга: прямой рендеринг в кадровый буфер (FRAME_BUFFER), пиксельный буфер (PIXEL_BUFFER) и FBO (FRAME_BUFFER_OBJECT). Класс osg::Camera предоставляет метод setRenderTargetImplementation() для задания цели рендеринга, например

```cpp
camera->setRenderTargetImplementation(osg::Camera::FRAME_BUFFER);
```

## Пример реализации рендеринга в текстуру

Для демострации техники рендеринга в текстуру реализуем такую задачку: создадим квадрат, натянем на него квадратную же текстуру, а в текстуру выполним рендеринг анимированной сцены, конечно же с полюбившейся нам цессоной. Программа, реализующая пример вышла достаточно объмной. Однако всё равно приведу её полный исходный текст.

**main.h**
```cpp
#ifndef		MAIN_H
#define		MAIN_H

#include    <osg/Camera>
#include    <osg/Texture2D>
#include    <osg/MatrixTransform>
#include    <osgDB/ReadFile>
#include    <osgGA/TrackballManipulator>
#include    <osgViewer/Viewer>

#endif
```

**main.cpp**
```cpp
#include	"main.h"

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
osg::Geometry *createQuad(const osg::Vec3 &pos, float w, float h)
{
    osg::ref_ptr<osg::Vec3Array> vertices = new osg::Vec3Array;
    vertices->push_back( pos + osg::Vec3( w / 2, 0.0f, -h / 2) );
    vertices->push_back( pos + osg::Vec3( w / 2, 0.0f,  h / 2) );
    vertices->push_back( pos + osg::Vec3(-w / 2, 0.0f,  h / 2) );
    vertices->push_back( pos + osg::Vec3(-w / 2, 0.0f, -h / 2) );

    osg::ref_ptr<osg::Vec3Array> normals = new osg::Vec3Array;
    normals->push_back(osg::Vec3(0.0f, -1.0f, 0.0f));

    osg::ref_ptr<osg::Vec2Array> texcoords = new osg::Vec2Array;
    texcoords->push_back( osg::Vec2(1.0f, 1.0f) );
    texcoords->push_back( osg::Vec2(1.0f, 0.0f) );
    texcoords->push_back( osg::Vec2(0.0f, 0.0f) );
    texcoords->push_back( osg::Vec2(0.0f, 1.0f) );

    osg::ref_ptr<osg::Geometry> quad = new osg::Geometry;
    quad->setVertexArray(vertices.get());
    quad->setNormalArray(normals.get());
    quad->setNormalBinding(osg::Geometry::BIND_OVERALL);
    quad->setTexCoordArray(0, texcoords.get());
    quad->addPrimitiveSet(new osg::DrawArrays(GL_QUADS, 0, 4));

    return quad.release();
}

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
int main(int argc, char *argv[])
{
    (void) argc; (void) argv;

    osg::ref_ptr<osg::Node> sub_model = osgDB::readNodeFile("../data/cessna.osg");

    osg::ref_ptr<osg::MatrixTransform> transform1 = new osg::MatrixTransform;
    transform1->setMatrix(osg::Matrix::rotate(0.0, osg::Vec3(0.0f, 0.0f, 1.0f)));
    transform1->addChild(sub_model.get());

    osg::ref_ptr<osg::Geode> model = new osg::Geode;
    model->addChild(createQuad(osg::Vec3(0.0f, 0.0f, 0.0f), 2.0f, 2.0f));

    int tex_widht = 1024;
    int tex_height = 1024;

    osg::ref_ptr<osg::Texture2D> texture = new osg::Texture2D;
    texture->setTextureSize(tex_widht, tex_height);
    texture->setInternalFormat(GL_RGBA);
    texture->setFilter(osg::Texture2D::MIN_FILTER, osg::Texture2D::LINEAR);
    texture->setFilter(osg::Texture2D::MAG_FILTER, osg::Texture2D::LINEAR);

    model->getOrCreateStateSet()->setTextureAttributeAndModes(0, texture.get());    

    osg::ref_ptr<osg::Camera> camera = new osg::Camera;
    camera->setViewport(0, 0, tex_widht, tex_height);
    camera->setClearColor(osg::Vec4(1.0f, 1.0f, 1.0f, 1.0f));
    camera->setClearMask(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

    camera->setRenderOrder(osg::Camera::PRE_RENDER);
    camera->setRenderTargetImplementation(osg::Camera::FRAME_BUFFER_OBJECT);
    camera->attach(osg::Camera::COLOR_BUFFER, texture.get());

    camera->setReferenceFrame(osg::Camera::ABSOLUTE_RF);
    camera->addChild(transform1.get());

    osg::ref_ptr<osg::Group> root = new osg::Group;
    root->addChild(model.get());
    root->addChild(camera.get());

    osgViewer::Viewer viewer;
    viewer.setSceneData(root.get());
    viewer.setCameraManipulator(new osgGA::TrackballManipulator);
    viewer.setUpViewOnSingleScreen(0);

    camera->setProjectionMatrixAsPerspective(30.0, static_cast<double>(tex_widht) / static_cast<double>(tex_height), 0.1, 1000.0);

    float dist = 100.0f;
    float alpha = 10.0f * 3.14f / 180.0f;
    
    osg::Vec3 eye(0.0f, -dist * cosf(alpha), dist * sinf(alpha));
    osg::Vec3 center(0.0f, 0.0f, 0.0f);
    osg::Vec3 up(0.0f, 0.0f, -1.0f);
    camera->setViewMatrixAsLookAt(eye, center, up);

    float phi = 0.0f;
    float delta = -0.01f;

    while (!viewer.done())
    {
        transform1->setMatrix(osg::Matrix::rotate(static_cast<double>(phi), osg::Vec3(0.0f, 0.0f, 1.0f)));
        viewer.frame();
        phi += delta;
    }

    return 0;
}
```

Для создания квадрата напишем отдельную свободную функцию

```cpp
osg::Geometry *createQuad(const osg::Vec3 &pos, float w, float h)
{
    osg::ref_ptr<osg::Vec3Array> vertices = new osg::Vec3Array;
    vertices->push_back( pos + osg::Vec3( w / 2, 0.0f, -h / 2) );
    vertices->push_back( pos + osg::Vec3( w / 2, 0.0f,  h / 2) );
    vertices->push_back( pos + osg::Vec3(-w / 2, 0.0f,  h / 2) );
    vertices->push_back( pos + osg::Vec3(-w / 2, 0.0f, -h / 2) );

    osg::ref_ptr<osg::Vec3Array> normals = new osg::Vec3Array;
    normals->push_back(osg::Vec3(0.0f, -1.0f, 0.0f));

    osg::ref_ptr<osg::Vec2Array> texcoords = new osg::Vec2Array;
    texcoords->push_back( osg::Vec2(1.0f, 1.0f) );
    texcoords->push_back( osg::Vec2(1.0f, 0.0f) );
    texcoords->push_back( osg::Vec2(0.0f, 0.0f) );
    texcoords->push_back( osg::Vec2(0.0f, 1.0f) );

    osg::ref_ptr<osg::Geometry> quad = new osg::Geometry;
    quad->setVertexArray(vertices.get());
    quad->setNormalArray(normals.get());
    quad->setNormalBinding(osg::Geometry::BIND_OVERALL);
    quad->setTexCoordArray(0, texcoords.get());
    quad->addPrimitiveSet(new osg::DrawArrays(GL_QUADS, 0, 4));

    return quad.release();
}
```

Функция принимает на вход позицию центра квадрата и его геометрические размеры. Далее создается массив вершин, массив нормалей и текстурных координат, после чего созданная геометрия возвращается из функции.

В теле основной программы загрузим модельку цессны

```cpp
osg::ref_ptr<osg::Node> sub_model = osgDB::readNodeFile("../data/cessna.osg");
```

Для того чтобы анимировать эту модель, создадим и проинициализируем трансформацию поворота вокруг оси Z

```cpp
osg::ref_ptr<osg::MatrixTransform> transform1 = new osg::MatrixTransform;
transform1->setMatrix(osg::Matrix::rotate(0.0, osg::Vec3(0.0f, 0.0f, 1.0f)));
transform1->addChild(sub_model.get());
```

Теперь создадим модель для основной сцены - квадрат на который будем выполнять рендеринг

```cpp
osg::ref_ptr<osg::Geode> model = new osg::Geode;
model->addChild(createQuad(osg::Vec3(0.0f, 0.0f, 0.0f), 2.0f, 2.0f));
```

Создаем пустую текстуру для квадрата размером 1024х1024 пикселя с форматом пикселя RGBA (32-битный трехкомпонентный цвет с альфа-каналом)

```cpp
int tex_widht = 1024;
int tex_height = 1024;

osg::ref_ptr<osg::Texture2D> texture = new osg::Texture2D;
texture->setTextureSize(tex_widht, tex_height);
texture->setInternalFormat(GL_RGBA);
texture->setFilter(osg::Texture2D::MIN_FILTER, osg::Texture2D::LINEAR);
texture->setFilter(osg::Texture2D::MAG_FILTER, osg::Texture2D::LINEAR);
```

Применяем эту текстуру к модели квадрата

```cpp
model->getOrCreateStateSet()->setTextureAttributeAndModes(0, texture.get()); 
```

Затем создаем камеру, которая будет запекать текстуру

```cpp
osg::ref_ptr<osg::Camera> camera = new osg::Camera;
camera->setViewport(0, 0, tex_widht, tex_height);
camera->setClearColor(osg::Vec4(1.0f, 1.0f, 1.0f, 1.0f));
camera->setClearMask(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
```

Вьюпорт камеры по размеру совпадает с размером текстуры. Кроме того не забываем задать цвет форна при очистке экрана и маску очистки, указывая очищать как буфер цвета, так и буфер глубины. Далее настраиваем камеру на рендеринг в текстуру

```cpp
camera->setRenderOrder(osg::Camera::PRE_RENDER);
camera->setRenderTargetImplementation(osg::Camera::FRAME_BUFFER_OBJECT);
camera->attach(osg::Camera::COLOR_BUFFER, texture.get());
```

Порядок рендеринга PRE_RENDER указывает на то, что рендеринг этой камерой выполняется до рендеринга в основную сцену. В качестве цели рендеренга указываем FBO и прикрепляем к камере нашу текстуру. Теперь настраиваем камеру на работу в абсолютной системе координат, а в качестве сцены задаем наше поддерево, которое мы жеоаем рендерить в текстуру: трансформация поворота с прикрепленной к ней моделькой цессны

```cpp
camera->setReferenceFrame(osg::Camera::ABSOLUTE_RF);
camera->addChild(transform1.get());
```

Создаем корневой групповой узел, добавляя в него основную модель (квадрат) и камеру обрабатывающую текстуру

```cpp
osg::ref_ptr<osg::Group> root = new osg::Group;
root->addChild(model.get());
root->addChild(camera.get());
```

Создаем и настраиваем вьювер

```cpp
osgViewer::Viewer viewer;
viewer.setSceneData(root.get());
viewer.setCameraManipulator(new osgGA::TrackballManipulator);
viewer.setUpViewOnSingleScreen(0);
```

Натраиваем матрицк проекции для камеры - перспективная проекция через параметры пирамицы отсечения

```cpp
camera->setProjectionMatrixAsPerspective(30.0, static_cast<double>(tex_widht) / static_cast<double>(tex_height), 0.1, 1000.0);
```

Настраиваем матрицу вида, задающую положение камеры в пространстве по отношению к началу координат подсцены с цессной

```cpp
float dist = 100.0f;
float alpha = 10.0f * 3.14f / 180.0f;

osg::Vec3 eye(0.0f, -dist * cosf(alpha), dist * sinf(alpha));
osg::Vec3 center(0.0f, 0.0f, 0.0f);
osg::Vec3 up(0.0f, 0.0f, -1.0f);
camera->setViewMatrixAsLookAt(eye, center, up);
```

Наконец, анимируем и отображаем сцену, меняя угол поворота самолета вокруг оси Z на каждом кадре

```cpp
float phi = 0.0f;
float delta = -0.01f;

while (!viewer.done())
{
    transform1->setMatrix(osg::Matrix::rotate(static_cast<double>(phi), osg::Vec3(0.0f, 0.0f, 1.0f)));
    viewer.frame();
    phi += delta;
}
```

В итоге мы получаем довольно интересную картинку

![](https://habrastorage.org/webt/se/g1/jr/seg1jrflbtmzfkb5cbtwys1hleg.gif)

