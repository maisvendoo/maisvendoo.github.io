---
layout: post
title:  "Введение в OpenSceneGraph: Трассировка, уведомления, логирование"
date:   2018-10-29 21:50:00 +0300
categories: jekyll update
---

OpenSceneGraph имеет механизм уведомлений, позволяющий выводить отладочные сообщения в процессе выполнения рендеринга, а так же инициированные разработчиком. Это серьезное подстпорье при трассировке и отладке программы. Система уведомлений OSG поддерживает вывод диагностической информации (ошибки, предупреждения, уведомления) на уровне ядра движка и плагинов к нему. Разработчик может вывести диагностическое сообщение в процессе работы программы, воспользовавшись функцией osg::notify().

<!--more-->

Данная функция работает как стандартный поток вывода стандартной бибиотеки C++ через перегрузку оператора <<. В качестве аргумента она принимает уровень сообщения: ALWAYS, FATAL, WARN, NOTICE, INFO, DEBUG_INFO и DEBUG_FP. Например

```cpp
osg::notify(osg::WARN) << "Some warning message" << std::endl;
```
выводит предупреждение с определенным пользователем текстом.

## Перенаправление уведомлений

Уведомления OSG могут содержать важную информацию о состоянии программыЮ расширениях графической подсистемы компьютера, возможных проблемах при работе движка. 

В некоторых случаях требуется выводить эти данные не в консоль, а иметь возможность перенаправить данный вывод в файл (в виде лога) либо на любой другой интерфейс, в том числе и графический виджет. Движок содержит специальный класс osg::NotifyHandler обеспечивающий перенаправление уведомленй в нужный разработчику поток вывода.

На простом примере рассмотрим, каким образом можно перенаправить вывод уведомлений, скажем, в текстовый файл лога. Напишем следующий код

**main.h**
```cpp
#ifndef     MAIN_H
#define     MAIN_H

#include    <osgDB/ReadFile>
#include    <osgViewer/Viewer>
#include    <fstream>

#endif  //  MAIN_H
```

**main.cpp**
```cpp
#include    "main.h"

class LogFileHandler : public osg::NotifyHandler
{
public:

    LogFileHandler(const std::string &file)
    {
        _log.open(file.c_str());
    }

    virtual ~LogFileHandler()
    {
        _log.close();
    }

    virtual void notify(osg::NotifySeverity severity, const char *msg)
    {
        _log << msg;
    }

protected:

    std::ofstream   _log;
};

int main(int argc, char *argv[])
{
    osg::setNotifyLevel(osg::INFO);
    osg::setNotifyHandler(new LogFileHandler("../logs/log.txt"));

    osg::ArgumentParser args(&argc, argv);
    osg::ref_ptr<osg::Node> root = osgDB::readNodeFiles(args);

    if (!root)
    {
        OSG_FATAL << args.getApplicationName() << ": No data loaded." << std::endl;
        return -1;
    }

    osgViewer::Viewer viewer;
    viewer.setSceneData(root.get());

    return viewer.run();
}
```

Для перенаправления вывода напишем класс LogFileHandler, являющийся наследником osg::NotifyHandler. Конструктор и деструктор этого класса управляют открытием и закрытием потока вывода _log, с которым связывается текстовый файл. Метод notify() есть аналогичный петод базового класса, переопределенный нами для вывода в файл уведомлений, передаваемых OSG в процессе работы через параметр msg.

**Класс LogFileHandler**
```cpp
class LogFileHandler : public osg::NotifyHandler
{
public:

    LogFileHandler(const std::string &file)
    {
        _log.open(file.c_str());
    }

    virtual ~LogFileHandler()
    {
        _log.close();
    }

    virtual void notify(osg::NotifySeverity severity, const char *msg)
    {
        _log << msg;
    }

protected:

    std::ofstream   _log;
};
```

Далее, в основной программе выполняем необходимые настройки

```cpp
osg::setNotifyLevel(osg::INFO);
```
устанавливаем уровень уведомлений.