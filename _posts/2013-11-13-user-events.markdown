---
layout: post
title:  "Введение в OpenSceneGraph: Создание и обработки пользовательских событий"
date:   2018-11-13 13:01:00 +0300
categories: jekyll update
---

OSG использует внутреннюю очередь событий (FIFO). События находящиеся в начале очереди обрабатываются и удаляются из неё. Вновь сгенерированные события помещаются в конец очереди. Метод handle() каждого обработчика события будет выполнятся столько раз, сколько событий находится в очереди. Очередь событий описывается классом osgGA::EventQueue, среди прочего позволяющий поместить событие в очередь в любой момент времени, вызовом метода addEvent(). Аргументом этого метода является указатель на osgGA::GUIEventAdapter, который можно настроить на определенное поведение методами setEventType() и т.д. 

Одним из методов класса osgGA::EventQueue является userEvent(), который задает пользовательское событие, связывая его с пользовательскими данными, указатель на которые передается ему в качестве параметра. Эти данные могут быть использованы для представления любого кастомного события.

Невозможно создать собственный экземпляр очереди событий. Этот экземпляр уже создан и прикреплен к экземпляру вьювера, так что можно только получить указатель на данный синглтон

```cpp
viewer.getEventQueue()->userEvent(data);
```

Пользовательские данные - объект наследника от osg::Referenced, то есть на него можно создать умный указатель.

При получении пользовательского события разработчик может извлеч из него данные вызовом метода getUserData() и обрабатывать их по своему усмотрению.

## Реализация пользовательского таймера

Многие библиотеки и фреймворки, реализующие GUI предоставляют разработчику класса для реализации таймеров, генерирующих событие по истечении определенного временного интервала. OSG не соделжит штатных средсв реализации таймеров, поэтому попробуем реализовать некое подобие таймера самостоятельно, используя интерфейс для создания пользовательских событий.

На что мы можем опереться при решении данной задачи? На некое периодическое событие, постоянно генерируемое рендером, например на FRAME - собитие отрисовки очередного кадра. Воспользуемся для этого все тем же примером с переключением модельки цессны с нормальной на горящую.

**main.h**
```cpp
#ifndef		MAIN_H
#define		MAIN_H

#include    <osg/Switch>
#include    <osgDB/ReadFile>
#include    <osgGA/GUIEventHandler>
#include    <osgViewer/Viewer>
#include    <iostream>

#endif
```

**main.cpp**
```cpp
#include	"main.h"

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
struct TimerInfo : public osg::Referenced
{
    TimerInfo(unsigned int c) : _count(c) {}
    unsigned int _count;
};

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
class TimerHandler : public osgGA::GUIEventHandler
{
public:

    TimerHandler(osg::Switch *sw, unsigned int interval = 1000)
        : _switch(sw)
        , _count(0)
        , _startTime(0.0)
        , _interval(interval)
        , _time(0)
    {

    }

    virtual bool handle(const osgGA::GUIEventAdapter &ea, osgGA::GUIActionAdapter &aa);

protected:

    osg::ref_ptr<osg::Switch> _switch;
    unsigned int _count;
    double _startTime;
    unsigned int _interval;
    unsigned int _time;
};

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
bool TimerHandler::handle(const osgGA::GUIEventAdapter &ea, osgGA::GUIActionAdapter &aa)
{
    switch (ea.getEventType())
    {
    case osgGA::GUIEventAdapter::FRAME:
    {

        osgViewer::Viewer *viewer = dynamic_cast<osgViewer::Viewer *>(&aa);

        if (!viewer)
            break;

        double time = viewer->getFrameStamp()->getReferenceTime();
        unsigned int delta = static_cast<unsigned int>( (time - _startTime) * 1000.0);
        _startTime = time;

        if ( (_count >= _interval) || (_time == 0) )
        {
            viewer->getEventQueue()->userEvent(new TimerInfo(_time));
            _count = 0;
        }

        _count += delta;
        _time += delta;

        break;
    }

    case osgGA::GUIEventAdapter::USER:

        if (_switch.valid())
        {
            const TimerInfo *ti = dynamic_cast<const TimerInfo *>(ea.getUserData());

            std::cout << "Timer event at: " << ti->_count << std::endl;

            _switch->setValue(0, !_switch->getValue(0));
            _switch->setValue(1, !_switch->getValue(1));
        }

        break;

    default:

        break;
    }

    return false;
}

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
int main(int argc, char *argv[])
{
    (void) argc; (void) argv;

    osg::ref_ptr<osg::Node> model1 = osgDB::readNodeFile("../data/cessna.osg");
    osg::ref_ptr<osg::Node> model2 = osgDB::readNodeFile("../data/cessnafire.osg");

    osg::ref_ptr<osg::Switch> root = new osg::Switch;
    root->addChild(model1.get(), true);
    root->addChild(model2.get(), false);

    osgViewer::Viewer viewer;
    viewer.setSceneData(root.get());
    viewer.addEventHandler(new TimerHandler(root.get(), 1000));
    
    return viewer.run();
}
```

Для начала определимся с форматом данных, пересылаемых в пользовательском сообщении, определив его как структуру

```cpp
struct TimerInfo : public osg::Referenced
{
    TimerInfo(unsigned int c) : _count(c) {}
    unsigned int _count;
};
```

Параметр _count будет содержать целое число милисекунд, прошедшее с момента запуска программы на момент получения очередного события таймера. Структура наследуется от класса osg::Referenced, чтобы ей можно было управлять через умные указатели OSG. Теперь создаем обработчик событий

```cpp
class TimerHandler : public osgGA::GUIEventHandler
{
public:

    TimerHandler(osg::Switch *sw, unsigned int interval = 1000)
        : _switch(sw)
        , _count(0)
        , _startTime(0.0)
        , _interval(interval)
        , _time(0)
    {

    }

    virtual bool handle(const osgGA::GUIEventAdapter &ea, osgGA::GUIActionAdapter &aa);

protected:

    osg::ref_ptr<osg::Switch> _switch;
    unsigned int _count;
    double _startTime;
    unsigned int _interval;
    unsigned int _time;
};
```

Этот обработчик имеет несколько специфических защищенных членов. Переменная _switch указывает на узел, переключающий модели самолета; _count - относительный отсчет времени, прошедшего с последней генерации события таймера, служит для отсчета временных интервалов; _startTime - временная переменная для храниния предыдущего отсчета времени, ведущегося вьювером; _time - общее время работы программы в милисекундах. Конструктор класса принимает в качестве параметра узел-переключатель а так же, опционально, требуемый временной интервал срабатывания таймера переключения.

В этом классе мы переопределяем метод handle()

```cpp
bool TimerHandler::handle(const osgGA::GUIEventAdapter &ea, osgGA::GUIActionAdapter &aa)
{
    switch (ea.getEventType())
    {
    case osgGA::GUIEventAdapter::FRAME:
    {

        osgViewer::Viewer *viewer = dynamic_cast<osgViewer::Viewer *>(&aa);

        if (!viewer)
            break;

        double time = viewer->getFrameStamp()->getReferenceTime();
        unsigned int delta = static_cast<unsigned int>( (time - _startTime) * 1000.0);
        _startTime = time;

        if ( (_count >= _interval) || (_time == 0) )
        {
            viewer->getEventQueue()->userEvent(new TimerInfo(_time));
            _count = 0;
        }

        _count += delta;
        _time += delta;

        break;
    }

    case osgGA::GUIEventAdapter::USER:

        if (_switch.valid())
        {
            const TimerInfo *ti = dynamic_cast<const TimerInfo *>(ea.getUserData());

            std::cout << "Timer event at: " << ti->_count << std::endl;

            _switch->setValue(0, !_switch->getValue(0));
            _switch->setValue(1, !_switch->getValue(1));
        }

        break;

    default:

        break;
    }

    return false;
}
```

Здесь мы анализируем тип принятого сообщения. Если это FRAME, то выполняются следующие действия:

1. Получаем указатель на вьювер

```cpp
osgViewer::Viewer *viewer = dynamic_cast<osgViewer::Viewer *>(&aa);
```

2. При получении корректного указателя, считываем время, прошедшее с момента запуска программы

```cpp
double time = viewer->getFrameStamp()->getReferenceTime();
```

вычисляем промежуток времени, затраченный на рендеринг кадра в милисекундах

```cpp
unsigned int delta = static_cast<unsigned int>( (time - _startTime) * 1000.0);
```

и запоминаем текущий временной отсчет

```cpp
_startTime = time;
```

Если значение счетчика _count превысило требуемый временной интервал (или это первый вызов когда _time ещё равно нулю), мы помещаем пользовательской сообщение в очередь, передав в определенной выше структуре время работы программы в милисекундах. Счетчик _count сбрасываем в ноль

```cpp
if ( (_count >= _interval) || (_time == 0) )
{
    viewer->getEventQueue()->userEvent(new TimerInfo(_time));
    _count = 0;
}
```

В не зависимости от значения _count мы должны нарастить его и _time на величину задержки, потребовавшейся на рисование кадра

```cpp
_count += delta;
_time += delta;
```

Так будет устроена генерация события таймера. Обработка события реализована так

```cpp
case osgGA::GUIEventAdapter::USER:

        if (_switch.valid())
        {
            const TimerInfo *ti = dynamic_cast<const TimerInfo *>(ea.getUserData());

            std::cout << "Timer event at: " << ti->_count << std::endl;

            _switch->setValue(0, !_switch->getValue(0));
            _switch->setValue(1, !_switch->getValue(1));
        }

        break;
```

Здесь мы проверяем на валидность указатель на перключающий узел, вычитываем данные из события, приводя из к структуре TimerInfo, выводим содержимое структуры на экран и переключаем состояние узла.

Код в функции main() аналогичен коду в предыдущих двух примерах с переключением, с той разницей что в данном случае мы вешаем на вьювер обработчик событий

```cpp
viewer.addEventHandler(new TimerHandler(root.get(), 1000));
```

передавая конструктору обработчика указатель на узел root и требуемый интервал переключения в милисекундах. Запустив пример мы увидим, что модели переключаются с интервалом в секунду, а в консоли обнаружим вывод моментов времени, в которые произошло переключение

```
Timer event at: 0
Timer event at: 1000
Timer event at: 2009
Timer event at: 3017
Timer event at: 4025
Timer event at: 5033
```

Пользовательское событие может быть сгенерировано в любой момент выполнения программы, а не только при приеме события FRAME, и это дает очень гибкий механизм обмена данными между частями программы, дает возможность обработки сигналов с нестандартных устройств ввода, таких как джойстики или перчатки VR, например.

