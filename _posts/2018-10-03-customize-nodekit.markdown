---
layout: post
title:  "Введение в OpenSceneGraph: Настройка поведения узлов сцены"
date:   2018-11-03 23:00:00 +0300
categories: jekyll update
---

Важным приемом работы с OSG является переопределение метода traverse(). Этот метод вызывается каждый раз, когда происходит отрисовка кадра. Они принимает параметр типа osg::NodeVisitor& который сообщает, какой проход графа сцены выполняется в данный момент (обновление, обработка событий или отсечение). Большинство узлов OSG переопределяют этот метод для реализации своего функционала.

При этом следует помнить, что переопределение метода traverse() может быть опасным, поскольку оно влияет на процесс обхода графа сцены и может привести к неправильному отображению сцены. Это так же неудобно, если вы желаете добавить новый функционал к нескольким типам узлов. В этом случае импользуют обратные вызовы узлов, разговор о которых пойдет при рассмотрении анимации объектов сцены.

Мы уже знаем, что узел osg::Switch может управлять отображением своих дочерных узлов, включая отображение одних узлов и выключая отображение других. Но он не умеет делать этого автоматически, поэтому создадим новый узел на базе старого, который будет переключатся между дочерними узлами в разные моменты времени, в соотвествии со значением внутренного счетчика.

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
class AnimatingSwitch : public osg::Switch
{
public:

    AnimatingSwitch() : osg::Switch(), _count(0) {}

    AnimatingSwitch(const AnimatingSwitch &copy, const osg::CopyOp &copyop = osg::CopyOp::SHALLOW_COPY) :
        osg::Switch(copy, copyop), _count(copy._count) {}

    META_Node(osg, AnimatingSwitch);

    virtual void traverse(osg::NodeVisitor &nv);

protected:

    unsigned int _count;
};

void AnimatingSwitch::traverse(osg::NodeVisitor &nv)
{
    if (!((++_count) % 60) )
    {
        setValue(0, !getValue(0));
        setValue(1, !getValue(1));
    }

    osg::Switch::traverse(nv);
}

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
int main(int argc, char *argv[])
{
    (void) argc; (void) argv;

    osg::ref_ptr<osg::Node> model1 = osgDB::readNodeFile("../data/cessna.osg");
    osg::ref_ptr<osg::Node> model2 = osgDB::readNodeFile("../data/cessnafire.osg");

    osg::ref_ptr<AnimatingSwitch> root = new AnimatingSwitch;
    root->addChild(model1.get(), true);
    root->addChild(model2.get(), false);

    osgViewer::Viewer viewer;
    viewer.setSceneData(root.get());

    return viewer.run();
}
```

Разберем этот пример по полочкам. Мы создаем новый класс AnimatingSwitch, который наследует от osg::Switch. 


```cpp
class AnimatingSwitch : public osg::Switch
{
public:

    AnimatingSwitch() : osg::Switch(), _count(0) {}

    AnimatingSwitch(const AnimatingSwitch &copy, const osg::CopyOp &copyop = osg::CopyOp::SHALLOW_COPY) :
        osg::Switch(copy, copyop), _count(copy._count) {}

    META_Node(osg, AnimatingSwitch);

    virtual void traverse(osg::NodeVisitor &nv);

protected:

    unsigned int _count;
};

void AnimatingSwitch::traverse(osg::NodeVisitor &nv)
{
    if (!((++_count) % 60) )
    {
        setValue(0, !getValue(0));
        setValue(1, !getValue(1));
    }

    osg::Switch::traverse(nv);
}
```

Этот класс содержит конструктор по-умолчанию

```cpp
AnimatingSwitch() : osg::Switch(), _count(0) {}
``` 

и конструктор для копирования, созданный в соотвествии с требованиями OSG

```cpp
AnimatingSwitch(const AnimatingSwitch &copy, const osg::CopyOp &copyop = osg::CopyOp::SHALLOW_COPY) :
        osg::Switch(copy, copyop), _count(copy._count) {}
```

Конструктор для копирования должен содержать в качестве параметров: константную ссылку на экземпляр класса, подлежащий копированию и параметра osg::CopyOp, задающий настройки копирования класса. Далее следуют довольно странные письмена

```cpp
META_Node(osg, AnimatingSwitch);
```

Это макрос, формирующий необходимую для наследника класса, производного от osg::Node структуру. Пока не придаем значения этому макросу - важно что он должен присутствовать при наследовании от osg::Switch при определении всех классов-потомков. Класс содержит защищенное поле _count - тот самый счетчик, на основе которого мы выполняем переключение. Переключение реализуем при переопределении метода traverse()

```cpp
void AnimatingSwitch::traverse(osg::NodeVisitor &nv)
{
    if (!((++_count) % 60) )
    {
        setValue(0, !getValue(0));
        setValue(1, !getValue(1));
    }

    osg::Switch::traverse(nv);
}
```

Переключение статуса отображения узлов будет происходить каждый раз, когда значение счетчика (инкрементируемого каждый вызов метода) будет кратно 60. Компилируем пример и запускаем его

![](https://habrastorage.org/webt/z5/fl/2u/z5fl2uhbnzxafceoro_wbga-pk4.gif)

Поскольку метод traverse() постоянно переопределяется для различных типов узлов, он должен предоставлять механихм для получения матриц преобразования и состояния рендера для дальнейшего использования их реализуемом перегрузкой алгоритме. Входной параметра osg::NodeVisitor является ключом к различным операциям с узлами. Он, в частности, указывает на тип текущего обхода графа сцены, таких как обновление, обработка событий и отсечение невидимых граней. Первые два связаны с обратными вызовами узлов и будут рассмотрены при изучении анимации.

Проход отсечения может быть идентифицирован путем преобразования объекта osg::NodeVisitor к osg::CullVisitor 

```cpp

osgUtil::CullVisitor *cv = dynamic_cast<osgUtil::CullVisitor *>(&nv);

if (cv)
{
	/// Выполняем что-то тут, характерное для обработки отсечения
}
```
