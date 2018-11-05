---
layout: post
title:  "Введение в OpenSceneGraph: Оконный и композитный режимы отображения"
date:   2018-11-05 21:51:00 +0300
categories: jekyll update
---

## Отображение сцены в оконном режиме

Класс osgViewer::Viewer может быть очень быстро перенастроен на отображение в оконном режиме. Как вы заметили, все предыдущие наши примеры отображались в полноэкранном режиме. Для переключения вьювера в оконный режим существует метод setUpViewInWindow(), который принимает в качестве параметров координаты левого верхнего угла окна, его ширину и высоту в пикселях

```cpp
viewer.setUpViewInWindow(50, 50, 800, 600);
```

Опциоанльно этот метод принимает пятый параметр - номер экрана, на который следует выводить окно, для того случая, если у вас не один монитор. Наверняка работая с несколькими мониторами в Windows вы наблюдали, что сцена расползается на все мониторы в полноэкранном режиме (в Linux такое не наблюдается).

Кроме того, в настройках проекта можно задать переменную окружения OSG_WINDOW таким образом

![](https://habrastorage.org/webt/-5/yg/ge/-5yggeqzivtjkymngxq3xk318qm.png)

что будет эквивалентно вызову setUpViewInWindow(), который в таком случае можно и не выполнять.

![](https://habrastorage.org/webt/2z/z_/px/2zz_pxckvywsn1c3hpebbwwj77e.png)

Для явного указания экрана, на который следует выводить вьювер в полноэкранном режиме можно воспользоваться методом setUpViewOnSingleScreen() указав в качестве параметра номер экрана (по-умолчанию 0). 

OSG поддерживает также демонстрационные сферические дисплеи. Для настройки отображения на таком дисплее можно использвать метод setUpViewFor3DSphericalDisplay().

## Композитный вьювер

Класс osgViewer::Viewer управляет одним видом, отображающим один граф сцены. Кроме него существует класс osgViewer::CompositeViewer который поддерживает несколько видов и несколько сцен. Он имеет те же методы run(), frame() и done() для управления процессом рендеринга, но при этом позволяет добавлять и удалять независимые виды с помощью методов addView() и removeView(), а также получение видов по их индексу методом getView(). Объект вида описывается классом osgViewer::View.

Класс osgViewer::View является базовым для класса osgViewer::Viewer. Он позволяет добалять корневой узел с данными сцены, манипулятор камеры и обработчики событий. Основное отличие этого класса (вид) от класса вьювера - он не позволяет рендерить сцену вызовами run() или frame(). Типичный сценарий добавления вида выглядит так

```cpp
osgViewer::CompositeViewer multiviewer;
multiviewer.addView( view );
```

Композитный вьювер позволяет отображать одну сцену в разных ракурсах, обображая эти ракурсы в разных окнах. Он так же позволяет отображать в разных окнах независимые сцены. Напишем простой пример использования композитного вьювера

**main.h**
```cpp
#ifndef		MAIN_H
#define		MAIN_H

#include    <osgDB/ReadFile>
#include    <osgViewer/CompositeViewer>

#endif
```

**main.cpp**
```cpp
#include	"main.h"

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
osgViewer::View *createView(int x, int y, int w, int h,
                            osg::Node *scene)
{
    osg::ref_ptr<osgViewer::View> view = new osgViewer::View;
    view->setSceneData(scene);
    view->setUpViewInWindow(x, y, w, h);

    return view.release();
}

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
int main(int argc, char *argv[])
{
    (void) argc; (void) argv;

    osg::ref_ptr<osg::Node> model1 = osgDB::readNodeFile("../data/cessna.osg");
    osg::ref_ptr<osg::Node> model2 = osgDB::readNodeFile("../data/cow.osg");
    osg::ref_ptr<osg::Node> model3 = osgDB::readNodeFile("../data/glider.osg");

    osgViewer::View *view1 = createView(50, 50, 320, 240, model1);
    osgViewer::View *view2 = createView(380, 50, 320, 240, model2);
    osgViewer::View *view3 = createView(185, 330, 320, 240, model3);

    osgViewer::CompositeViewer viewer;
    viewer.addView(view1);
    viewer.addView(view2);
    viewer.addView(view3);
    
    return viewer.run();
}
```

Создание отдельного вида поместим в функцию, принимающую в качестве параметров положение и размеры окна, а также сцену в виде указателя на её корневой узел

```cpp
osgViewer::View *createView(int x, int y, int w, int h,
                            osg::Node *scene)
{
    osg::ref_ptr<osgViewer::View> view = new osgViewer::View;
    view->setSceneData(scene);
    view->setUpViewInWindow(x, y, w, h);

    return view.release();
}
```

Здесь мы создаем вид, управляемый умным указателем на объект osgViewer::View

```cpp
osg::ref_ptr<osgViewer::View> view = new osgViewer::View;
```

задаем данные отображаемой сцены и оконный режим отображения в окне с заданным положением и размерами

```cpp
view->setSceneData(scene);
view->setUpViewInWindow(x, y, w, h);
```

Вид возвращаем из функции по правилам возврата умных указателей

```cpp
return view.release();
```

Теперь в основной программе загружаем три разных модели

```cpp
osgViewer::View *view1 = createView(50, 50, 320, 240, model1);
osgViewer::View *view2 = createView(380, 50, 320, 240, model2);
osgViewer::View *view3 = createView(185, 330, 320, 240, model3);
```

создаем три разных вида

```cpp
osgViewer::View *view1 = createView(50, 50, 320, 240, model1);
osgViewer::View *view2 = createView(380, 50, 320, 240, model2);
osgViewer::View *view3 = createView(185, 330, 320, 240, model3);
```

создаем композитный вьювер и добавляем в него созданные ранее виды

```cpp
osgViewer::CompositeViewer viewer;
viewer.addView(view1);
viewer.addView(view2);
viewer.addView(view3);
```

и запускаем рендеринг абсолютно аналогично тому, как мы это делали в случае с одной сценой

```cpp
return viewer.run();
```

Всё! При запуске программы мы получим три разный окна. Содержимым каждого окна можно управлять независимо. Любое из окон можно закрыть стандартным способом, а из приложения целиком выйти по нажатию Esc.

![](https://habrastorage.org/webt/dh/na/kl/dhnaklox5z-apbaynehnea_okss.png)

