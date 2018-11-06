---
layout: post
title:  "Введение в OpenSceneGraph: Инкапсуляция машины состояний OpenGL"
date:   2018-11-06 08:53:00 +0300
categories: jekyll update
---

Как правило, работая с параметрами рендеринга, OpenGL действует как конечный автомат. Состояние рендеринга - это совокупность атрибутов состояния, таких как источники света, материалы, текстуры и режимы отображения, включаемые и выключаемые функциями glEnable() и glDisable(). При установке определенного состояния оно действует до тех пор, пока какая-либо дргуая функция не изменит его. Ковейер OpenGL поддерживает стек состояний для сохранения и восстановления состояний в любой момент времени. Машина состояний предоставляет разработчику полный контроль над текущими и сохраненными в стеке состояниями рендеринга.

Однако такой подход неудобен при работе с OSG. По этой причине машина состояний OpenGL инкапсулирована в классе osg::StateSet, который берет на себя операции по работе со стеком состояний, их установке в процессе обхода графа сцены.

Экземпляр класса osg::StateSet содержит подмножество различных состояний рендеринга и может применять их к узлам сцены osg::Node и отрисовываем геометрическим объектам osg::Drawable с помощью метода setStateSet()

```cpp
osg::StateSet *stateset = new osg::StateSet;
node->setStateSet(stateset);
```

Более безопасным способом будет использование метода getOrCreateStateSet(), который гарантирует возврат корректного состояния и присоединение его к узлу или отрисовываемому объекту

```cpp
osg::StateSet *stateset = node->getOrCreateStateSet();
```

Классы osg::Node и osg::Drawable управляют переменной-членом osg::StateSet через умный указатель osg::ref_ptr<>. Это означает, что набор состояний может разделятся между несколькими объектами сцены и будет уничтожен только при уничтожении всех этих объектов.

## Атрибуты и режимы

В OSG определен класс osg::StateAttribute для хранения атрибутов рендеринга. Это виртуальный базовый класс, который наследуется различными атрибутами рендеринга, такимим как свет, материал и туман.

Режимы рендеринга работаю как переключатели, которые можно включать и выключать. Кроме того, с ними связаны перечислители, которые используют для указания типа режима OpenGL. Иногда режим рендеринга ассоциируется с атрибутом, например режим GL_LIGHTING включает переменные для источников света, отправляемые в конвейер OpenGL когда включен, и выключает освещение в противном случае.

Класс osg::StateSet делит атрибуты и режимы на две групы: текстурные и не текстурные. Он имеет несколько публичных методов для добавления не текстурных атрибутов и режимов в набор состояний:

1. setAttribute() - добавляет объект типа osg::StateAttribute к набору состояний. Атрибуты одного типа не могут сосуществовать в одном наборе состояний. Предыдущее заданное значение будет перезаписано новым.
2. setMode() - прикрепляет перечислитель режима к набору состояний и устанавливает его зачение в osg::StateAttribute::ON или в osg::StateAttribute::OFF, что обозначает включение или отключение режима.
3. setAttributeAndModes() - прикрепляет отрибут рендеринга и ассоциированный с ним режим и задает зачение переключателя (по-умолчанию ON). Слудет иметь в виду, что не каждый атрибут имеет соответствующий режим, но вы можете использовать этот метод в любом случае.

Для установки атрибута и ассоциированного с ним режима можно использовать такой код

```cpp
stateset->setAttributeAndModes(attr, osg::StateAttribute::ON);
```

Для установки текстурных аттрибутов необходимо передавать дополнительный параметр, чтобы указать текстуру, к которой он должен быть применен. Для этго osg::StateSet предоставляет несколько других публичных методов, таких как setTextureAttribute(), setTextureMode() и setTextureAttributeAndModes()

```cpp
stateset->setTextureAttributeAndModes(0, textattr, osg::StateAttribute::ON);
```

применяет атрибут textattr к теустуре с идентификатором 0.

## Задание режима отображения полигонов для узлов сцены

Проилюстрируем вышеописанную теорию практическим примером - изменением режима растеризации полигонов OpenGL, используя класс osg::PolygonMode, наследующий от osg::StateAttribute. Этот класс инкапсулирует функцию glPolygonMode() и предоставляет интерфейс для установки режима отображения полигонов для конкретного узла сцены.

**main.h**
```cpp
#ifndef		MAIN_H
#define		MAIN_H

#include    <osg/PolygonMode>
#include    <osg/MatrixTransform>
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

    osg::ref_ptr<osg::Node> model = osgDB::readNodeFile("../data/cessna.osg");

    osg::ref_ptr<osg::MatrixTransform> transform1 = new osg::MatrixTransform;
    transform1->setMatrix(osg::Matrix::translate(-25.0f, 0.0f, 0.0f));
    transform1->addChild(model.get());

    osg::ref_ptr<osg::MatrixTransform> transform2 = new osg::MatrixTransform;
    transform2->setMatrix(osg::Matrix::translate(25.0f, 0.0f, 0.0f));
    transform2->addChild(model.get());

    osg::ref_ptr<osg::PolygonMode> pm = new osg::PolygonMode;
    pm->setMode(osg::PolygonMode::FRONT_AND_BACK, osg::PolygonMode::LINE);
    transform1->getOrCreateStateSet()->setAttribute(pm.get());

    osg::ref_ptr<osg::Group> root = new osg::Group;
    root->addChild(transform1.get());
    root->addChild(transform2.get());

    osgViewer::Viewer viewer;
    viewer.setSceneData(root.get());
    
    return viewer.run();
}
```

Здесь мы загрзуим модель нашей любимой цессны и применяя к ней трансформации получим два экземпляра модели. К одному из них, тому что слева, применим аттрибут, задающий режим каркасного отображения полигонов

```cpp
osg::ref_ptr<osg::PolygonMode> pm = new osg::PolygonMode;
pm->setMode(osg::PolygonMode::FRONT_AND_BACK, osg::PolygonMode::LINE);
transform1->getOrCreateStateSet()->setAttribute(pm.get());
```

![](https://habrastorage.org/webt/pg/v5/ew/pgv5ewaulc_tfjztw5h8gyyap1i.png)

Если обратится к спецификации OpenGL, то можно легко представить себе какие параметры отображения полигонов будут доступны нам при использовании setMode() в данном конкретном случае. Первый параметр может принимать значения osg::PolygonMode::FRONT, BACK и FRONT_AND_BACK, соотвествующие перечислителям OpenGL GL_FRONT, GL_BACK, GL_FRONT_AND_BACK. Второй параметр модет принимать значения osg::PolygonMode::POINT, LINE и FILL, которые соответствуют GL_POINT, GL_LINE и GL_FILL. Никаких других трюков, как это часто бывает при разработке на чистом OpenGL тут применять не нужно - OSG берет на себя большую часть работы. Режим отображения полигонов не имеет связанного режима и не требует вызова пары glEnable()/glDisable(). Метод setAttributeAndModes() будет прекрасно работать и в данном случае, но значение его третьего параметра будет при этом бесполезным.