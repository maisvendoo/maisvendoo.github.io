---
layout: post
title:  "Введение в OpenSceneGraph: Применение паттерна посетитель"
date:   2018-11-04 14:16:00 +0300
categories: jekyll update
---

Паттерн посетитель испльзуется для доступа к операциям по изменению элементов графа сцены без изменения классов этих элементов. Класс-посетитель реализует все соответствующие виртуальные функции для применения их к различным типам элементам через механизм двойной диспетчеризации. Используя этот механизм разработчик может создать свой экземпляр посетителя, реализовав нужный ему функционал с помощью специальных операторов и привязать посетителя к различным типам элементов графа сцены "на лету", не меняя функционала самих элементов. Это отличный способ расширить функциональность элемента без определения подклассов этих элементов.

Для реализации данного механизма в OSG определен класс osg::NodeVisitor. Класс, унаследованный от osg::NodeVisitor перемещается по графй сцены, посещает каждый узел и применяет к нему определенные разработчиком операции. Это основной класс, используемый для вмешательство в процесс обновления узлов и отсечения невидимых узлов, а так же пнименения некоторых других операций, связанных с модификацией геометрии узлов сцены, таких как osgUtil::SmoothingVisitor, osgUtil::Simplifier и osgUtil::TriStripVisitor.

Для создания подкласса посетителя, мы должны переопределить один или несколько виртуальных перегружаемых методов apply(), предоставляемых базовым классом osg::NodeVisitor. Эти методы имеются у большинства основных типов узлов OSG. Посетитель автоматически вызовет метод apply() для каждого из посещенный при обходе графа сцены узлов. Разработчик переопределяет метод apply() для каждого из необходимых ему типов узлов.

В реализации метода apply() разработчик, в соотвествующий момент, должен вызвать метод traverse() базового класса osg::NodeVisitor. Это инициирует переход посетителя к следующему узлу, либо дочернему, либо соседнему по уровню иерархии, если текущий узел не имеет дочерних узлов, на которые можно осуществить переход. Отсутствие вызова traverse() означает остановку обхода графа сцены и оставшаяся часть графа сцены игнорируется. 

Перегрузки метода apply() имеют унифицированные форматы

```cpp
virtual void apply( osg::Node& );
virtual void apply( osg::Geode& );
virtual void apply( osg::Group& );
virtual void apply( osg::Transform& );
```

Чтобы обойти подграф текущего узла, для объекта-посетителя, необходимо задать режим обхода, например так

```cpp
ExampleVisitor visitor;
visitor->setTraversalMode( osg::NodeVisitor::TRAVERSE_ALL_CHILDREN );
node->accept( visitor );
```

Режим обхода задается несколькими перечеслителями

1. TRAVERSE_ALL_CHILDREN - перемещение по вем дочерним узлам.
2. TRAVERSE_PARENTS - проход назад от текущего узла, не доходя до корневого узла
3. TRAVERSE_ACTIVE_CHILDREN - обход исключительно активных узлов, то есть тех, видимость которых активирована через узел osg::Switch.

## Анализ сруктуры горящей цессны

Разработчик всегда может проанализировать ту часть графа сцены, что порождается моделью, загруженной из файла.

**main.h**
```cpp
#ifndef		MAIN_H
#define		MAIN_H

#include    <osgDB/ReadFile>
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
class InfoVisitor : public osg::NodeVisitor
{
public:

    InfoVisitor() : _level(0)
    {
        setTraversalMode(osg::NodeVisitor::TRAVERSE_ALL_CHILDREN);
    }

    std::string spaces()
    {
        return std::string(_level * 2, ' ');
    }

    virtual void apply(osg::Node &node);

    virtual void apply(osg::Geode &geode);

protected:

    unsigned int _level;
};

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
void InfoVisitor::apply(osg::Node &node)
{
    std::cout << spaces() << node.libraryName() << "::"
              << node.className() << std::endl;

    _level++;
    traverse(node);
    _level--;
}

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
void InfoVisitor::apply(osg::Geode &geode)
{
    std::cout << spaces() << geode.libraryName() << "::"
              << geode.className() << std::endl;

    _level++;

    for (unsigned int i = 0; i < geode.getNumDrawables(); ++i)
    {
        osg::Drawable *drawable = geode.getDrawable(i);

        std::cout << spaces() << drawable->libraryName() << "::"
                  << drawable->className() << std::endl;
    }

    traverse(geode);
    _level--;
}

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
int main(int argc, char *argv[])
{
    osg::ArgumentParser args(&argc, argv);
    osg::ref_ptr<osg::Node> root = osgDB::readNodeFiles(args);

    if (!root.valid())
    {
        OSG_FATAL << args.getApplicationName() << ": No data leaded. " << std::endl;
        return -1;
    }
    
    InfoVisitor infoVisitor;
    root->accept(infoVisitor);

    osgViewer::Viewer viewer;
    viewer.setSceneData(root.get());

    return viewer.run();
}
```



```
osg::Group
  osg::MatrixTransform
    osg::Geode
      osg::Geometry
      osg::Geometry
    osg::MatrixTransform
      osgParticle::ModularEmitter
      osgParticle::ModularEmitter
  osgParticle::ParticleSystemUpdater
  osg::Geode
    osgParticle::ParticleSystem
    osgParticle::ParticleSystem
    osgParticle::ParticleSystem
    osgParticle::ParticleSystem
```
