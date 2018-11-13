---
layout: post
title:  "Введение в OpenSceneGraph: Окна, графический контекст и камеры"
date:   2018-11-13 16:00:00 +0300
categories: jekyll update
---

Мы уже говорили о том, что класс osg::Camera управляет связанным с ним графическим контекстом OpenGL. Графический контекст инкапсулирует информацию о том, как и куда происходит отрисовка объектов и какие атрибуты состояния к ним применяются. Под контекстом понимают графическое окно, вернее его клиентскую область, или пиксельный буфер OpenGL, который хранит данные пикселей без передачи их в кадровый буфер.

OSG использует класс osg::GraphicsContext для представления абстрактного графического контекста, и класс osg::GraphicsWindow, для представления абстрактного графического окна. Последния имеет метод getEventQueue() для управления событиями от элементов GUI. Вообще говоря графический контекст есть платформоспецифичное понятие, поэтому большую часть работы по сосзданию окна и связыванию его контекста с контекстом OpenGL, OSG берет на себя. При вызове метода createGraphicsContext() класса osg::GraphicsContext() требуемый код (а его не мало, поверьте!) будет сгенерирован препроцессором автоматически, в зависимости от платформы. От нас лишь требуется передать этому методу аргумент типа osg::GraphicsContex::Traits, содержащий описание того, какое окно мы хотим получить.

## Класс osg::GraphicsContext::Traits

Слово "traits" в переводе с английского означает "черты". Так вот, вышеупомянутый класс описывает черты будущего окна, и содержит все свойства для описания графического контекста. Он отличается от класса osg::DisplaySettings, который управляет характеристиками всех графических контекстов для вновь созданных камер. Основные публичные свойства этого класса перечислены в таблице ниже

|-------------------|-----------------------------|---------------------|----------------------------------------------------------------|
|Атрибут класса     |Тип                          |Значение по-умолчанию|Описание                                                        |
|-------------------|-----------------------------|---------------------|----------------------------------------------------------------|
|x                  |int                          |0                    |Начальная горизонтальная позиция окна                           |
|y                  |int                          |0                    |Начальная вертикальная позиция окна                             |
|width              |int                          |0                    |Ширина окна                                                     |
|height             |int                          |0                    |Высота окна                                                     |
|windowName         |std::string                  |""                   |Заголовок окна                                                  |
|windowDecoration   |bool                         |false                |Флаг отображения заголовка окна                                 |
|red                |unsigned int                 |8                    |Число бит красного в буфере цвета OpenGL                        |
|green              |unsigned int                 |8                    |Число бит зеленого в буфере цвета OpenGL                        |
|blue               |unsigned int                 |8                    |Число бит синего в буфере цвета OpenGL                          |
|alpha              |unsigned int                 |8                    |Число бит в альфа-буфере OpenGL                                 |
|depth              |unsigned int                 |24                   |Число бит в буфере глубины OpenGL                               |
|stencil            |unsigned int                 |0                    |Число бит в буфере трафарета OpenGL                             |
|doubleBuffer       |bool                         |false                |Использвать двойной буфер                                       |
|samples            |unsigned int                 |0                    |Число примитивом сглаживания                                    |
|quadBufferStereo   |bool                         |false                |Использовать счетверенный стерео-буфер (для оборудования NVidia)|
|inheritedWindowData|osg::ref_ptr<osg::Referenced>|NULL                 |Описатель данных, ассоциированных с окном                       |


Для инициализации объекта Traits необходимо выполнить следующий код

```cpp
osg::ref_ptr<osg::GraphicsContext::Traits> traits = new osg::GraphicsContext::Traits;

traits->x = 50;
traits->y = 100;
...
```

## Настраиваем окно приложения OSG

Чтобы создать окно с заданными характеристиками, необходимо проделать следующие шаги:

1. Настроить объект типа osg::GraphicsContext::Traits
2. Создать графический контекст окна
3. Связать этот графический контекст с камерой
4. Сделать камеру основной для вьювера


**main.h**
```cpp
#ifndef		MAIN_H
#define		MAIN_H

#include    <osg/GraphicsContext>
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

    osg::ref_ptr<osg::GraphicsContext::Traits> traits = new osg::GraphicsContext::Traits;
    traits->x = 50;
    traits->y = 50;
    traits->width = 800;
    traits->height = 600;
    traits->windowName = "OSG application";
    traits->windowDecoration = true;
    traits->doubleBuffer = true;
    traits->samples = 4;

    osg::ref_ptr<osg::GraphicsContext> gc = osg::GraphicsContext::createGraphicsContext(traits.get());

    osg::ref_ptr<osg::Camera> camera = new osg::Camera;
    camera->setGraphicsContext(gc);
    camera->setViewport( new osg::Viewport(0, 0, traits->width, traits->height) );
    camera->setClearMask(GL_DEPTH_BUFFER_BIT | GL_COLOR_BUFFER_BIT);
    camera->setClearColor( osg::Vec4(0.2f, 0.2f, 0.4f, 1.0f) );
    double aspect = static_cast<double>(traits->width) / static_cast<double>(traits->height);
    camera->setProjectionMatrixAsPerspective(30.0, aspect, 1.0, 1000.0);
    camera->getOrCreateStateSet()->setMode(GL_DEPTH_TEST, osg::StateAttribute::ON);

    osg::ref_ptr<osg::Node> root = osgDB::readNodeFile("../data/cessna.osg");

    osgViewer::Viewer viewer;
    viewer.setCamera(camera.get());
    viewer.setSceneData(root.get());
    
    return viewer.run();
}
```

Для задания настроек окна создаем экземпляр класса osg::GraphicsContext::Traits и инициализируем его необходимыми нам параметрами

```cpp
osg::ref_ptr<osg::GraphicsContext::Traits> traits = new osg::GraphicsContext::Traits;
traits->x = 50;
traits->y = 50;
traits->width = 800;
traits->height = 600;
traits->windowName = "OSG application";
traits->windowDecoration = true;
traits->doubleBuffer = true;
traits->samples = 4;
```

После этого создаем графический контекст, передавая в качестве настроек указатель на traits

```cpp
osg::ref_ptr<osg::GraphicsContext> gc = osg::GraphicsContext::createGraphicsContext(traits.get());
```

Создаем камеру

```cpp
osg::ref_ptr<osg::Camera> camera = new osg::Camera;
```

Связываем камеру с созданным графическим контекстом

```cpp
camera->setGraphicsContext(gc);
```

Настрииваем вьюпорт, задаем маску очистки буферов, задаем цвет очистки 

```cpp
camera->setViewport( new osg::Viewport(0, 0, traits->width, traits->height) );
camera->setClearMask(GL_DEPTH_BUFFER_BIT | GL_COLOR_BUFFER_BIT);
camera->setClearColor( osg::Vec4(0.2f, 0.2f, 0.4f, 1.0f) );
```

Настраиваем матрицу перспективной проекции

```cpp
double aspect = static_cast<double>(traits->width) / static_cast<double>(traits->height);
camera->setProjectionMatrixAsPerspective(30.0, aspect, 1.0, 1000.0);
```

Не забываем включить тест глубины, для корректного отображения граней

```cpp
camera->getOrCreateStateSet()->setMode(GL_DEPTH_TEST, osg::StateAttribute::ON);
```

Загружаем модель самолета

```cpp
osg::ref_ptr<osg::Node> root = osgDB::readNodeFile("../data/cessna.osg");
```

Настраиваем и запускаем вьювер, указав настроенную нами камеру в качетве основной камеры

```cpp
osgViewer::Viewer viewer;
viewer.setCamera(camera.get());
viewer.setSceneData(root.get());

return viewer.run();
```

На выходе иммем окно с требуемыми параметрами

![](https://habrastorage.org/webt/rs/av/d-/rsavd-z8aawrwpykioblv1kxsni.png)

Звголовок окна не отображается потому, что в настройках моего оконного менеджера эта функция выключена. Если запустить пример в Windows или Linux с другими настройками, то заголовок будет на своем месте.

К слову сказать, метод setUpViewInWindow(), который мы использовали ранее для запуска рендера в оконном режиме делает абсолютно то же самое, что мы проделали сейчас.

