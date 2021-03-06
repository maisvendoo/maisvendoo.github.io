---
layout: post
title:  "Введение в OpenSceneGraph: Техника фоновой загрузки узлов сцены"
date:   2018-11-03 00:40:00 +0300
categories: jekyll update
---

В движке OSG представлены классы osg::ProxyNode и osg::PagedLOD, предназначенный для баллансировки нагрузки при рендеринге сцены. Оба класса наследуются от osg::Group.

Узел типа osg::ProxyNode уменьшеает время запуска приложения до начала рендеринга, если в сцене огромное количество загружаемых с диска и отображаемых моделей. Он работае как интерфейс к внешним файлам, позволяя выполнять отложенную загрузку моделей. Для добавления дочерних узлов используется метод setFileName() (вместо addChild) чтобы установить имя файла модели на диске и загрузить его динамически.

Узел osg::PagedNode наследует методы osg::LOD и загружает и выгружает уровни детализации таким образом, чтобы избежать перегрузки конвейера OpenGL и обеспечить плавную отрисовку сцены.

## Динамическая (runtime) загрузка модели

Посмотрим, как происходит процесс загрузки модели с применением osg::ProxyNode.

**main.h**
```cpp
#ifndef		MAIN_H
#define		MAIN_H

#include    <osg/ProxyNode>
#include    <osgViewer/Viewer>

#endif
```

**main.cpp**
```cpp
#include	"main.h"

int main(int argc, char *argv[])
{
    (void) argc; (void) argv;

    osg::ref_ptr<osg::ProxyNode> root = new osg::ProxyNode;
    root->setFileName(0, "../data/cow.osg");

    osgViewer::Viewer viewer;
    viewer.setSceneData(root.get());
    
    return viewer.run();
}
```

Процесс загрузки здесь немного отличается

```cpp
osg::ref_ptr<osg::ProxyNode> root = new osg::ProxyNode;
root->setFileName(0, "../data/cow.osg");	
```

Вместо явной загрузки модели коровы мы указываем корневой ноде имя файла, где содержится модель и индекс дочерней ноды, куда следует поместить эту модель после её загрузки. При выполнении программы мы получи такой результат

![](https://habrastorage.org/webt/4u/z_/yb/4uz_ybvtau_okn8gtbsyvesuqo8.png)

Видно, что точка обзора выбрана не лучшим образом - камера уприается прямо в зеркальный бок коровы. Это произошло потому, что модель загрузилась уже после запуска рендера и инициализации камеры, когда нода 0 ещё не была видна. Вьювер просто не смог просчитать необходимые параметры камеры. Однако, модель загрузилась и мы может настроить режим её отображения путем манипуляций мышью

![](https://habrastorage.org/webt/kj/qp/hm/kjqphmpi5-thkedfpuqflwfywoa.png)

Что происходит в рассмотренном примере? osg::ProxyNode и osg::PagedLOD работают в данном случае как контейнеры. Внутренний менеджер данных OSG будет послылать запросы и загружать данные в граф сцены по мере того, как будут становится доступны новые пути к файлам и уровни детализациию. 

Данный механизм работает в нескольких фоновых потоках и управляет загрузкой статических данных, расположенных в файлах на диске и динамических данных, генерируемых и добавляемых в процессе выполнения программы.

Движок автоматически обрабатывает узлы, не отображаемые в текущем вьюпорту и удаляет их из графа сцены когда рендер перегружен. Однако, такое поведение не затрагивает узлы osg::ProxyNode.

Как и прокси-узел, класаа osg::PagedLOD также имеет метод setFileName() для задания пути к загружаемой модели, однако для его необходимо установить диапазон дистанции видимости, как для узла osg::LOD. При условии, что у нас имеется файл cessna.osg и низкополигональная модель уровня L1 мы можем организовать выгружаемую ноду следующим образом

```cpp
osg::ref_ptr<osg::PagedLOD> pagedLOD = new osg::PagedLOD;
pagedLOD->addChild(modelL1, 200.0f, FLT_MAX );
pagedLOD->setFileName( 1, "cessna.osg" );
pagedLOD->setRange( 1, 0.0f, 200.0f );
```

Нужно поинмать, что узел modelL1 не может быть выгружен из памяти, так как это обычный дочерний не прокси-узел.

При рендеринге внешне не видна разница между osg::LOD и osg::PagedLOD, если использовать только один уровень детализации модели. Интересной идеей будет организовать громадный кластер моделей Cessna, используя класс osg::MatrixTransform. Для этого можно использовать например такую функцию

```cpp
osg::Node* createLODNode( const osg::Vec3& pos )
{
	osg::ref_ptr<osg::PagedLOD> pagedLOD = new osg::PagedLOD;
	…
	osg::ref_ptr<osg::MatrixTransform> mt = new osg::MatrixTransform;
	mt->setMatrix( osg::Matrix::translate(pos) );
	mt->addChild( pagedLOD.get() );
	return mt.release();
}
```

Пример программы реализующей фоновую загрузку 10000 самолетов

**main.h**
```cpp
#ifndef		MAIN_H
#define		MAIN_H

#include    <osg/PagedLOD>
#include    <osg/MatrixTransform>
#include    <osgViewer/Viewer>

#endif
```

**main.cpp**
```cpp
#include	"main.h"

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
osg::Node *createLODNode(const std::string &filepath, const osg::Vec3 &pos)
{
    osg::ref_ptr<osg::PagedLOD> pagedLOD = new osg::PagedLOD;
    pagedLOD->setFileName(0, filepath);
    pagedLOD->setRange(0, 0.0f, FLT_MAX);

    osg::ref_ptr<osg::MatrixTransform> mt = new osg::MatrixTransform;
    mt->setMatrix(osg::Matrix::translate(pos));
    mt->addChild(pagedLOD.get());

    return mt.release();
}

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
int main(int argc, char *argv[])
{
    (void) argc; (void) argv;

    osg::ref_ptr<osg::Group> root = new osg::Group;

    float dist = 50.0f;
    int N = 100;

    for (int i = 0; i < N; ++i)
    {
        float x = i * dist;

        for (int j = 0; j < N; ++j)
        {
            float y = j * dist;
            osg::Vec3 pos(x, y, 0.0f);
            osg::ref_ptr<osg::Node> node = createLODNode("../data/cessna.osg", pos);
            root->addChild(node.get());
        }
    }

    osgViewer::Viewer viewer;
    viewer.setSceneData(root.get());

    return viewer.run();
}
```

Предполагается, что самолеты будут располагаться на плоскости с интервалом в 50 единиц координат. При загрузке мы увидим, что загружаются только те цесны, что попадаю в кадр. Те самолеты, что изчезают из кадра пропадают из дерева сцены.

![](https://habrastorage.org/webt/zx/cc/j4/zxccj4cjwh5a0ejfpl6mjudq8zi.png)

