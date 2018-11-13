---
layout: post
title:  "Введение в OpenSceneGraph: Проигрывание видео на текстуре"
date:   2018-11-12 22:36:00 +0300
categories: jekyll update
---

Было бы круто, если бы могли проводить видеоконференции в воображаемом трехмерном мире. Нет ничего невозможного -- создаем прямоугольный полигон в качестве экрана и крепим к нему динамическую текстуру. Текстура содержит серию изображений, составляющих видео. OSG использует класс osg::ImageStream, работающий с потоком изображений, получая их из видеокамеры или Интернет-трансляции. На сегодняшний день OSG имеет несколько готовых к работе плагинов, поддерживающих загрузку и воспроизведение видео в форматах AVI, MPG, MOV и так далее.

Пока мы рассмотрим этот эффект на более простом примере, используя другой класс osg::ImageSequence, позволяющий сохранять несколько объектов изображений с последовательным их воспроизведением. Этот класс имеет следующие публичные методы:

1. addImage() -- добавление изображения osg::Image в последовательность. Кроме того доступны методы setImage() и getImage() с доступом к последовательности по индексу, а так же getNumImages(), для получения числа изображений в последовательности.
2. addImageFile() и setImageFile() -- выполняют те же действия, но в качестве источника изображения указывается файл на диске.
3. setLength() -- задать общее время проигрывания последовательности в секундах.
4. setTimeMultiplier() -- задает множитель ускорения/замедления проигрывания последовательности.
5. play(), pause(), rewind() и seek() -- соотвественно начать проигрывание, поставить на паузу, перемотать и стать на заданную позицию (в секундах).

Покажем работу с этим классом на примере - создадим последовательность кадров, изображающих некий точечный источник света с изменяющейся яркостью и проиграем это импровизированное видео на текстуре.

**main.h**
```cpp
#ifndef		MAIN_H
#define		MAIN_H

#include    <osg/ImageSequence>
#include    <osg/Texture2D>
#include    <osg/Geometry>
#include    <osg/Geode>
#include    <osgViewer/Viewer>

#endif
```

**main.cpp**
```cpp
#include	"main.h"

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
osg::Image *createSpotLight(const osg::Vec4 &centerColor,
                            const osg::Vec4 &bgColor,
                            int size,
                            float power)
{
    osg::ref_ptr<osg::Image> image = new osg::Image;
    image->allocateImage(size, size, 1, GL_RGBA, GL_UNSIGNED_BYTE);

    float mid = (float(size) - 1) * 0.5f;
    float div = 2.0f / float(size);

    for (int r = 0; r < size; ++r)
    {
        unsigned char *ptr = image->data(0, r);

        for (int c = 0; c < size; ++c)
        {
            float dx = (float(c) - mid) * div;
            float dy = (float(r) - mid) * div;

            float r = powf(1.0f - sqrtf(dx*dx + dy*dy), power);

            if (r < 0.0f)
                r = 0.0f;

            osg::Vec4 color = centerColor * r + bgColor * (1.0f - r);
            *ptr++ = static_cast<unsigned char>((color[0]) * 255.0f);
            *ptr++ = static_cast<unsigned char>((color[1]) * 255.0f);
            *ptr++ = static_cast<unsigned char>((color[2]) * 255.0f);
            *ptr++ = static_cast<unsigned char>((color[3]) * 255.0f);
        }
    }

    return image.release();
}

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
int main(int argc, char *argv[])
{
    (void) argc; (void) argv;

    osg::Vec4 centerColor(1.0f, 1.0f, 0.0f, 1.0f);
    osg::Vec4 bgColor(0.0f, 0.0f, 0.0f, 1.0f);

    osg::ref_ptr<osg::ImageSequence> sequence = new osg::ImageSequence;
    sequence->addImage(createSpotLight(centerColor, bgColor, 64, 3.0f));
    sequence->addImage(createSpotLight(centerColor, bgColor, 64, 3.5f));
    sequence->addImage(createSpotLight(centerColor, bgColor, 64, 4.0f));
    sequence->addImage(createSpotLight(centerColor, bgColor, 64, 3.5f));

    osg::ref_ptr<osg::Texture2D> texture = new osg::Texture2D;
    texture->setImage(sequence.get());

    osg::ref_ptr<osg::Geode> geode = new osg::Geode;
    geode->addDrawable(osg::createTexturedQuadGeometry(osg::Vec3(),
                                                       osg::Vec3(1.0, 0.0, 0.0),
                                                       osg::Vec3(0.0, 0.0, 1.0)));
    geode->getOrCreateStateSet()->setTextureAttributeAndModes(0, texture.get(), osg::StateAttribute::ON);

    sequence->setLength(0.5);
    sequence->play();

    osgViewer::Viewer viewer;
    viewer.setSceneData(geode.get());
    
    return viewer.run();
}
```

Для реализации подобной анимации необходимо сгенерировать набор текстур с изображением светящейся точки. Для этого напишем отдельную функцию

```cpp
osg::Image *createSpotLight(const osg::Vec4 &centerColor,
                            const osg::Vec4 &bgColor,
                            int size,
                            float power)
{
    osg::ref_ptr<osg::Image> image = new osg::Image;
    image->allocateImage(size, size, 1, GL_RGBA, GL_UNSIGNED_BYTE);

    float mid = (float(size) - 1) * 0.5f;
    float div = 2.0f / float(size);

    for (int r = 0; r < size; ++r)
    {
        unsigned char *ptr = image->data(0, r);

        for (int c = 0; c < size; ++c)
        {
            float dx = (float(c) - mid) * div;
            float dy = (float(r) - mid) * div;

            float r = powf(1.0f - sqrtf(dx*dx + dy*dy), power);

            if (r < 0.0f)
                r = 0.0f;

            osg::Vec4 color = centerColor * r + bgColor * (1.0f - r);
            *ptr++ = static_cast<unsigned char>((color[0]) * 255.0f);
            *ptr++ = static_cast<unsigned char>((color[1]) * 255.0f);
            *ptr++ = static_cast<unsigned char>((color[2]) * 255.0f);
            *ptr++ = static_cast<unsigned char>((color[3]) * 255.0f);
        }
    }

    return image.release();
}
```

В качестве параметров функция будет принимать цвет источника, цвет фона, размер текстуры в пикселях (полагаем текстуру квадратной) и показатель мощности источника, на основании которого мы будем формировать его изображение. Чтобы не расписывать подробно алгоритм формирования изображения (оставлю это читателю для самостоятельного разбора), поясню только ключевые моменты. В функции мы создаем новое изображение и выделяем под него память в соответсвии с его размером и типом данных для отображения цветовых компонент

```cpp
osg::ref_ptr<osg::Image> image = new osg::Image;
image->allocateImage(size, size, 1, GL_RGBA, GL_UNSIGNED_BYTE);
```

Далее вычисляем цвет каждого пикселя как суммарный цвет доли цвета источника и цвета фона

```cpp
osg::Vec4 color = centerColor * r + bgColor * (1.0f - r);
```

где r - параметр смешения, экспоненциально зависящий от заданной мощности источника и расстояния от точки до центра текстуры (полагаем, что источник находится в центре текстуры)

```cpp
float r = powf(1.0f - sqrtf(dx*dx + dy*dy), power);
```

Полученный цвет назначаем соотвествующему пикселю текстуры путем непосредственного обращения к буферу данных изображения по указателю ptr

```cpp
*ptr++ = static_cast<unsigned char>((color[0]) * 255.0f);
*ptr++ = static_cast<unsigned char>((color[1]) * 255.0f);
*ptr++ = static_cast<unsigned char>((color[2]) * 255.0f);
*ptr++ = static_cast<unsigned char>((color[3]) * 255.0f);
```

который берем из объекта изображения

```cpp
unsigned char *ptr = image->data(0, r);
```

В функции main() задаем цвет источника и цвет фона

```cpp
osg::Vec4 centerColor(1.0f, 1.0f, 0.0f, 1.0f);
osg::Vec4 bgColor(0.0f, 0.0f, 0.0f, 1.0f);
```

Создаем объект последовательности изображений, заполняя его несколькими изображениями источника с разной яркостью

```cpp
osg::ref_ptr<osg::ImageSequence> sequence = new osg::ImageSequence;
sequence->addImage(createSpotLight(centerColor, bgColor, 64, 3.0f));
sequence->addImage(createSpotLight(centerColor, bgColor, 64, 3.5f));
sequence->addImage(createSpotLight(centerColor, bgColor, 64, 4.0f));
sequence->addImage(createSpotLight(centerColor, bgColor, 64, 3.5f));
```

Создаем текстуру из данной последовательности

```cpp
osg::ref_ptr<osg::Texture2D> texture = new osg::Texture2D;
texture->setImage(sequence.get());
```

Создаем квадратный полигон и применяем к нему данную текстуру

```cpp
osg::ref_ptr<osg::Geode> geode = new osg::Geode;
geode->addDrawable(osg::createTexturedQuadGeometry(osg::Vec3(),
                                                   osg::Vec3(1.0, 0.0, 0.0),
                                                   osg::Vec3(0.0, 0.0, 1.0)));
geode->getOrCreateStateSet()->setTextureAttributeAndModes(0, texture.get(), osg::StateAttribute::ON);
```

Задаем продолжительность проигрывания последовательности в полсекунды и запускаем проигрывание

```cpp
sequence->setLength(0.5);
sequence->play();
```

Инициализируем и запускаем вьювер (можно я не буду больше об этом писать?)

```cpp
osgViewer::Viewer viewer;
viewer.setSceneData(geode.get());

return viewer.run();
```

В итоге имеем интересный эффект динамической текстуры

![](https://habrastorage.org/webt/sn/nl/bf/snnlbf7fbtkcqxpstokpotk-cuk.gif)

Класс osg::ImageSequence обновляет данные рендеринга на каждом кадре на основе хранимых в нем изображений. 

Интересным является применение методов addFileName() и setFileName(), загружающих файлы изображений непосредственно с диска. Здесь возможно несколько режимов работы, задаваемых методом setMode():

1. PRE_LOAD_ALL_IMAGES -- загрузка всех изображений сразу. Экономит время на загрузку, но раздувает используемую память процесса.
2. PAGE_AND_RETAIN_IMAGES -- загрузка файлов на лету с последующим кэшированием данных изображений.
3. PAGE_AND_DISCARD_USED_IMAGES -- удаляет все использованные изображения из памяти и перезагружает их, когда они вновь потребуются.

Для задания режима следует выполнить вызов

```cpp
imageSequence->setMode( osg::ImageSequence::PAGE_AND_RETAIN_IMAGES );
```