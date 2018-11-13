---
layout: post
title:  "Введение в OpenSceneGraph: Взаимодейсвие с внешними интерфейсами - основы"
date:   2018-11-13 08:59:00 +0300
categories: jekyll update
---

OSG является авбстрактной графической библиотекой. С одной стороны она абстрагируется от процедурного интерфейса OpenGL, предоставляя разработчику набор классов, инкапсулирующих всю механику OpneGL API. С другой стороны, она абстрагируется и от конкретного графического интерфейса пользователя, так как подходы к его реализации различны для разных платформ и имеют особенности даже в рамках одной и той же платформы (MFC, Qt, .Net для Windows, например).

Не зависимо от платформы, с точки зрения приложения, взаимодействие пользователя с графическим интерфейсом сводится к генерации его элементами последовательности событий, обрабатываемых затем внутри приложения. Большинство графических фреймворков используют такой подход, однако даже впределах одной платформы они, к сожалению, несовместимы между собой.

По этой причине OSG предоставляет свой собственный базовый интерфейс для обработки событий виждетов графического интерфейса и пользовательского ввода на основе класса osgGA::GUIEventHandler. Этот обрабочик может быть прикреплен к вьюверу вызовом метода addEventHandler() и удален методом removeEventHandler(). Естесвенно, конкретный класс-обработчик должен быть унаследован от класса osgGA::GUIEventHandler, и в нем должен быть переопределен метод handle(). Этот метод принимает на вход два агрумента: osgGA::GUIEventAdapter, содержащий очередь событий от GUI и osg::GUIActionAdepter, используемый для обратной связи. Типичным, при определении является такая конструкция

```cpp
bool handle(const osgGA::GUIEventAdapter &ea, osgGA::GUIActionAdepter &aa)
{
	// Здесь выполняются конкретные операции по обработке событий
}
```  

Параметр osgGA::GUIActionAdapter позволяет разработчику попросить GUI выполнить некоторые действия, в ответ на событие. В большинстве случаев через этот параметр воздействуют на вьювер, указательна который может быть получен динамическим преобразованием указателя

```cpp
osgViewer::Viewer* viewer = dynamic_cast<osgViewer::Viewer *>(&aa);
```

## Обработка событий клавиатуры и мыши

Класс osgGA::GUIEventAdapter() управляет веми типами событий, поддерживаемых OSG, предоставляя данные для установки и получения их параметров. Метод getEventType() возвращает текущее событие GUI, содержащееся в очереди событий. Каждый раз, переопределяя метод handle() обработчика, при вызове этого методы следует использовать данный геттер для получения события и определения его типа.

Нижеследующая таблица описывает все доступные события

|------------------------|--------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
|Тип события             |Описание                                    |Методы получения данных события                                                                                                         |
|------------------------|--------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
|PUSH/RELEASE/DOUBLECLICK|Нажатие/Отпускаие и двойной клик кнопок мыши|getX(), getY() - получение позиции курсора. getButton() - код нажатой кнопки (LEFT_MOUSE_BUTTON, RIGHT_MOUSE_BUTTON, MIDDLE_MOUSE_BUTTON|
|SCROLL                  |Скролинг колесом(ами) мыши                  |getScrollingMotion() - возвращает значения SCROOL_UP, SCROLL_DOWN, SCROLL_LEFT, SCROLL_RIGHT                                            |
|DRAG                    |Перетаскивание мышью                        |getX(), getY() - позиция курсора; getButtonMask() - значения аналогичные getButton()                                                    |
|MOVE                    |Перемещение мыши                            |getX(), getY() - позиция курсора                                                                                                        |
|KEYDOWN/KEYUP           |Нажатие/Отпускание клавиши на клавиатуре    |getKey() - ASCII-код нажатой клавиши или значение перечислителя Key_Symbol (например KEY_BackSpace)                                     |
|FRAME                   |Событие, генерируемой при отрисовке кадра   |нет входных данных                                                                                                                      |
|USER                    |Событие, определяемое пользователем         |getUserDataPointer() - возвращает указатель на буфер пользовательских данных (буфер управляется умным указателем)                       |

Соществует также метод getModKeyMask() для получения информации о нажатой клавише-модификаторе (возвращает значения вида MODKEY_CTRL, MODKEY_SHIFT, MODKEY_ALT и так далее), позволяя обрабатывать комбинации клавиш, в которых используются модификаторы

```cpp
if (ea.getModKeyMask() == osgGA::GUIEventAdapter::MODKEY_CTRL)
{
	// Обработка нажатия клавиши Ctrl
}
```

Следует иметь ввиду, что методы-сеттеры типа setX(), setY(), setEventType() и т.п. не используются в обработчике handle(). Они вызываются низкоуровневой графической окнонной системой OSG для помещения собития в очередь.

## Управляем цессной с клавиатуры

Мы уже хорошо умеем трансформировать объекты сцены через классы osg::MatrixTransform. Мы рассмотрели различного рода анимации с помощью классов osg::AnimationPath и osg::Animation. Но для интерактивности приложения (например игрового) анимаций и трансформаций явно недостаточно. Следующим шаком будет управление положением объектов на сцене с устройств пользовательского ввода. Попробуем прикрутить к нашей любимой цессне управление.

**main.h**
```cpp
#ifndef		MAIN_H
#define		MAIN_H

#include    <osg/MatrixTransform>
#include    <osgDB/ReadFile>
#include    <osgGA/GUIEventHandler>
#include    <osgViewer/Viewer>

#endif
```

**main.cpp**
```cpp
#include	"main.h"

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
class ModelController : public osgGA::GUIEventHandler
{
public:

    ModelController( osg::MatrixTransform *node ) : _model(node) {}

    virtual bool handle(const osgGA::GUIEventAdapter &ea, osgGA::GUIActionAdapter &aa);

protected:

    osg::ref_ptr<osg::MatrixTransform> _model;
};

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
bool ModelController::handle(const osgGA::GUIEventAdapter &ea, osgGA::GUIActionAdapter &aa)
{
    (void) aa;

    if (!_model.valid())
        return false;

    osg::Matrix matrix = _model->getMatrix();

    switch (ea.getEventType())
    {
    case osgGA::GUIEventAdapter::KEYDOWN:
        switch (ea.getKey())
        {
        case 'a': case 'A':
            matrix *= osg::Matrix::rotate(-0.1, osg::Z_AXIS);
            break;

        case 'd': case 'D':
            matrix *= osg::Matrix::rotate( 0.1, osg::Z_AXIS);
            break;

        case 'w': case 'W':
            matrix *= osg::Matrix::rotate(-0.1, osg::X_AXIS);
            break;

        case 's': case 'S':
            matrix *= osg::Matrix::rotate( 0.1, osg::X_AXIS);
            break;

        default:

            break;
        }

        _model->setMatrix(matrix);

        break;

    default:

        break;
    }

    return true;
}

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
int main(int argc, char *argv[])
{
    (void) argc; (void) argv;

    osg::ref_ptr<osg::Node> model = osgDB::readNodeFile("../data/cessna.osg");

    osg::ref_ptr<osg::MatrixTransform> mt = new osg::MatrixTransform;
    mt->addChild(model.get());

    osg::ref_ptr<osg::Group> root = new osg::Group;
    root->addChild(mt.get());

    osg::ref_ptr<ModelController> mcontrol = new ModelController(mt.get());

    osgViewer::Viewer viewer;
    viewer.addEventHandler(mcontrol.get());
    viewer.getCamera()->setViewMatrixAsLookAt( osg::Vec3(0.0f, -100.0f, 0.0f), osg::Vec3(), osg::Z_AXIS );
    viewer.getCamera()->setAllowEventFocus(false);
    viewer.setSceneData(root.get());
    
    return viewer.run();
}
```

![](https://habrastorage.org/webt/a0/gu/od/a0guoddsokpfkthvefo07tfn1h4.gif)
