---
layout: post
title:  "Введение в OpenSceneGraph: Анимация объектов сцены"
date:   2018-11-08 16:45:00 +0300
categories: jekyll update
---

В предыдущей статье мы реализовали анимацию объекта сцены, путем изменения параметров его трасформации внутри цикла отрисовки сцены. Как уже многократно упоминалось, подобный подход таит в себе понтенциально опасное поведение приложение при многопоточном рендеринге. Для решения данной проблемы применяют механизм обратных вызовов, выполняемых при обходе графа сцены. 

## Обратные вызовы, реализованные в OSG

В движке существует несколько типов обратных вызовов. Обратные вызовы реализуются специальными классами, среди которых osg::NodeCallback предназначен для обрабоки процесса обновления узлов сцены, а osg::Drawable::UpdateCallback, osg::Drawable::EventCallback и osg::Drawable:CullCallback - выполняют те же функции, но для объектов геометрии.

Класс osg::NodeCallback имеет переопределяемый виртуальный оператор operator(), предоставляемый разработчку для реализации собственного функционала. Чтобы обратный вызов срабатывал, необходимо прикрепить экземпляр класса вызова к тому узлу, для который будет обрабатываться вызовом метода setUpdateCallback() или addUpdateCallback(). Оператор operator() автоматически вызывается во время обновления узлов графа сцены при рендеринге каждого кадра.

Нижеследующая таблица представляет перечень обратных вызовов, доступных разработчику в OSG

|------------------------------------------|-----------------------------|-----------------|----------------------------------------|
|Имя                                       |Функтор обратного вызова     |Виртуальный метод|Метод для присоединения к объекту       |
|------------------------------------------|-----------------------------|-----------------|----------------------------------------|
|Оновление узла                            |osg::NodeCallback            |operator()       |osg::Node::setUpdateCallback()          |
|Событие узла                              |osg::NodeCallback            |operator()       |osg::Node::setEventCallback()           |
|Отсечение узла                            |osg::NodeCallback            |operator()       |osg::Node::setCullCallback()            |
|Обновление геометрии                      |osg::Drawable::UpdateCallback|update()         |osg::Drawable::setUpdateCallback()      |
|Событие геометрии                         |osg::Drawable::EventCallback |event()          |osg::Drawable::setEventCallback()       |
|Отсечение геометрии                       |osg::Drawable::CullCallback  |cull()           |osg::Drawable::setCullCallback()        |
|Обновление атрибутов                      |osg::StateAttributeCallback  |operator()       |osg::StateAttribute::setUpdateCallback()|
|Событие атрибутов                         |osg::StateAttributeCallback  |operator()       |osg::StateAttribute::setEventCallback() |
|Общее обновление                          |osg::Uniform::Callback       |operator()       |osg::Uniform::setUpdateCallback()       |
|Общее событие                             |osg::Uniform::Callback       |operator()       |osg::Uniform::setEvevtCallback()        |
|Обратный вызов для камеры перед отрисовкой|osg::Camera::DarwCallback    |operator()       |osg::Camera::PreDrawCallback()          |
|Обратный вызов для камеры после отрисовки |osg::Camera::DarwCallback    |operator()       |osg::Camera::PostDrawCallback()         |

## Переключение osg::Switch при обновлении дерева сцены

Когда мы изучали класс osg::Switch, мы писали пример с переключением двух моделей самолетов. Теперь мы повторим этот пример, но сделаем всё правильно, используя механизм обратных вызовов OSG.

**main.h**
```cpp
#ifndef		MAIN_H
#define		MAIN_H

#include    <osg/Switch>
#include    <osgDB/ReadFile>
#include    <osgViewer/Viewer>

#endif
```

**main.cpp**
```cpp
#include	"main.h"

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
class SwitchingCallback : public osg::NodeCallback
{
public:

    SwitchingCallback() : _count(0) {}

    virtual void operator()(osg::Node *node, osg::NodeVisitor *nv);

protected:

    unsigned int _count;
};

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
void SwitchingCallback::operator()(osg::Node *node, osg::NodeVisitor *nv)
{
    osg::Switch *switchNode = static_cast<osg::Switch *>(node);

    if ( !((++_count) % 60) && switchNode )
    {
        switchNode->setValue(0, !switchNode->getValue(0));
        switchNode->setValue(1, !switchNode->getValue(0));
    }

    traverse(node, nv);
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
    root->addChild(model1, true);
    root->addChild(model2, false);

    root->setUpdateCallback( new SwitchingCallback );

    osgViewer::Viewer viewer;
    viewer.setSceneData(root.get());
    
    return viewer.run();
}
```

Необходимо создать класс, уноследовав его от osg::NodeCallback, управляющий узлом osg::Switch

```cpp
class SwitchingCallback : public osg::NodeCallback
{
public:

    SwitchingCallback() : _count(0) {}

    virtual void operator()(osg::Node *node, osg::NodeVisitor *nv);

protected:

    unsigned int _count;
};
```

Счетчик _count будет управлять переключением узла osg::Switch  с отображения одной дочерней ноды на другую, в зависимости от совего значения. В консрукторе мы инициализируем счетчик, а виртуальный метод operator() переопределяем

```cpp
void SwitchingCallback::operator()(osg::Node *node, osg::NodeVisitor *nv)
{
    osg::Switch *switchNode = static_cast<osg::Switch *>(node);

    if ( !((++_count) % 60) && switchNode )
    {
        switchNode->setValue(0, !switchNode->getValue(0));
        switchNode->setValue(1, !switchNode->getValue(0));
    }

    traverse(node, nv);
}
```

Узел, на котором сработал вызов передается в него через параметр node. Поскольку мы точно знаем, что это будет узел типа osg::Switch, мы выполняем статическое приведение указателя на node к указателю на узел-переключатель

```cpp
osg::Switch *switchNode = static_cast<osg::Switch *>(node);
``` 

Переключение обображаемых дочерних узлов будем выполнять при валидном значении этого указателя, и когда значение счетчика будет кратно 60

```cpp
if ( !((++_count) % 60) && switchNode )
{
    switchNode->setValue(0, !switchNode->getValue(0));
    switchNode->setValue(1, !switchNode->getValue(0));
}
```

Не забываем вызвать метод traverse() для продолжение рекурсивного обхода графа сцены

```cpp
traverse(node, nv);
```

Остальной код программы тривиален, за исключеним строчки

```cpp
root->setUpdateCallback( new SwitchingCallback );
```

где мы назначаем созданный нами обратный вызов узлу root с типом osg::Switch. Работа программы аналогична предыдущему примеру

![](https://habrastorage.org/webt/z5/fl/2u/z5fl2uhbnzxafceoro_wbga-pk4.gif)

В прошлый раз, для реализации данного поведения узла osg::Switch мы перепределяли метод traverse() у класса-наследника osg::Switch.

