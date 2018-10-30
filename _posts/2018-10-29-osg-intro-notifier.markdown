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
устанавливаем уровень уведомлений INFO, то есть вывод в лог всей информации о работе движка, включая текущие уведомления о нормальной работе.

```cpp
osg::setNotifyHandler(new LogFileHandler("../logs/log.txt"));
``` 
устанавливаем обработчик уведомлений. Далее обрабатываем аргрументы командной строки, в которых передаются пути к загружаемым моделям

```cpp
osg::ArgumentParser args(&argc, argv);
osg::ref_ptr<osg::Node> root = osgDB::readNodeFiles(args);

if (!root)
{
	OSG_FATAL << args.getApplicationName() << ": No data loaded." << std::endl;
	return -1;
}
```
При этом обрабатываем ситуацию отсутствия данных в командной строке, выводя сообщение в лог в ручном режиме посредством макроса OSG_FATAL. Запускаем программу со следующими аргументами

![](https://habrastorage.org/webt/mc/h4/so/mch4sot5pfjjpq9vb2dlolnjllm.png)

получая вывод в файл лога наподобие этого

```
Opened DynamicLibrary osgPlugins-3.7.0/mingw_osgdb_osgd.dll
CullSettings::readEnvironmentalVariables()
CullSettings::readEnvironmentalVariables()
Opened DynamicLibrary osgPlugins-3.7.0/mingw_osgdb_deprecated_osgd.dll
OSGReaderWriter wrappers loaded OK
CullSettings::readEnvironmentalVariables()
void StateSet::setGlobalDefaults()
void StateSet::setGlobalDefaults() ShaderPipeline disabled.
   StateSet::setGlobalDefaults() Setting up GL2 compatible shaders
CullSettings::readEnvironmentalVariables()
CullSettings::readEnvironmentalVariables()
CullSettings::readEnvironmentalVariables()
CullSettings::readEnvironmentalVariables()
ShaderComposer::ShaderComposer() 0xa5ce8f0
CullSettings::readEnvironmentalVariables()
ShaderComposer::ShaderComposer() 0xa5ce330
View::setSceneData() Reusing existing scene0xa514220
 CameraManipulator::computeHomePosition(0, 0)
    boundingSphere.center() = (-6.40034 1.96225 0.000795364)
    boundingSphere.radius() = 16.6002
 CameraManipulator::computeHomePosition(0xa52f138, 0)
    boundingSphere.center() = (-6.40034 1.96225 0.000795364)
    boundingSphere.radius() = 16.6002
Viewer::realize() - No valid contexts found, setting up view across all screens.
Applying osgViewer::ViewConfig : AcrossAllScreens
.
.
.
.
ShaderComposer::~ShaderComposer() 0xa5ce330
ShaderComposer::~ShaderComposer() 0xa5ce8f0
ShaderComposer::~ShaderComposer() 0xa5d6228
close(0x1)0xa5d3e50
close(0)0xa5d3e50
ContextData::unregisterGraphicsContext 0xa5d3e50
DatabasePager::RequestQueue::~RequestQueue() Destructing queue.
DatabasePager::RequestQueue::~RequestQueue() Destructing queue.
DatabasePager::RequestQueue::~RequestQueue() Destructing queue.
DatabasePager::RequestQueue::~RequestQueue() Destructing queue.
ShaderComposer::~ShaderComposer() 0xa5de4e0
close(0x1)0xa5ddba0
close(0)0xa5ddba0
ContextData::unregisterGraphicsContext 0xa5ddba0
Done destructing osg::View
DatabasePager::RequestQueue::~RequestQueue() Destructing queue.
DatabasePager::RequestQueue::~RequestQueue() Destructing queue.
DatabasePager::RequestQueue::~RequestQueue() Destructing queue.
DatabasePager::RequestQueue::~RequestQueue() Destructing queue.
Closing DynamicLibrary osgPlugins-3.7.0/mingw_osgdb_osgd.dll
Closing DynamicLibrary osgPlugins-3.7.0/mingw_osgdb_deprecated_osgd.dll
```

Не беда что в данный момент эта информация может показаться вам бессмысленной - в будущем подобный вывод может помочь отладить ошибки в вашей программе.

По-умолчанию OSG посылает сообщения в стандартный вывод std::cout и сообщения об ошибках в поток std::cerr. Однако, посредством переопределения обработчика уведомлений, как было показано в примере, этот вывод можно перенаправить в любой поток вывода, в том числе и в элементы графического интерфейса.

Следует помнить, что при установке высокого уровня уведомлений (например FATAL) система игнорирует все уведомления более низкого уровня. Например, в подобном случае

```cpp
osg::setNotifyLevel(osg::FATAL);
.
.
.
osg::notify(osg::WARN) << "Some message." << std::endl;
```

пользовательское сообщение просто не будет выведено.
