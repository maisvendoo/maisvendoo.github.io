---
layout: post
title:  "Введение в OpenSceneGraph: Понятие об интерполяции движения"
date:   2018-11-12 14:13:00 +0300
categories: jekyll update
---

Предположим, что некий поезд, идущий от станции A к станции B затрачивает на это перемещение 15 минут. Как можно смоделировать эту ситуацию, изменяя положение поезда в обратном вызове? Самый простой способ - соотнести положение станции A с моментом времени 0, а станции B - с моментом 15 минут и равномерно перемещать поезд между этими моментами времени. Такой простейший подход называется линейной интерполяцией. При линейной интерполяции вектор, задающий положение промежуточной точки описывается формулой

```
p = (1 - t) * p0 + t * p1
```

где p0 - начальная точка; p1 - конечная точка; t - параметр, изменяющийся равномерно от 0 до 1. Однако, движение поезда намного сложнее: выйдя со станции A он разгоняется, потом движется с постоянной скоростью, а затем замедляется, останавливаясь на станции B. Линейная интерполяция такой процесс уже не в состоянии описать и выглядит неестественно.

OSG предоставляет разработчику библиотеку osgAnimation, содержащую ряд стандартный алгоритмов интерполяции, применяемых для плавной анимации перемещения объектов сцены. Каждая из этих функций иммет обычно два аргрумента: начальное значение параметра (обычно 0) и конечное значение параметра (обычно 1). Эти функции могут быть применены к начальному участку движения (InMotion), к конечному участку (OutMotion) или к начальному и конечному участку движения (InOutMotion)

|-------------------------------|---------------|----------------|------------------|
|Тип движения                   |in-класс       |out-класс       |in/out-класс      |
|-------------------------------|---------------|----------------|------------------|
|Линейная интерполяция          |LinearMotion   |--              |--                |
|Квадратичная интерполяция      |InQuadMotion   |OutQuadMotion   |InOutQuadMotion   |
|Кубическая интерполяция        |InCubicMotion  |OutCubicMotion  |InOutCubicMotion  |
|Интерполяция 4-порядка         |InQuartMotion  |OutQuartMotion  |InOutQuartMotion  |
|Интерполяция с эффектом отскока|InBounceMotion |OutBounceMotion |InOutBounceMotion |
|Интерполяция с упругим отскоком|InElasticMotion|OutElasticMotion|InOutElasticMotion|
|Синусоидальная интерполяция    |InSineMotion   |OutSineMotion   |InOutSineMotion   |
|Интерполяция обратной функцией |InBackMotion   |OutBackMotion   |InOutBackMotion   |
|Круговая интерполяция          |InCircMotion   |OutCircMotion   |InOutCircMotion   |
|Экспоненциальная интерполяция  |InExpoMotion   |OutExpoMotion   |InOutExpoMotion   |


Для создания линейной интерполяции движения объекта пишем такой код

```cpp
osg::ref_ptr<osgAnimation::LinearMotion> motion = new osgAnimation::LinearMotion(0.0f, 1.0f);
```

## Анимирование узлов трансформации

Анимация движения по траектории - наиболее распространенный вид анимации в графических приложениях. Этот прием может быть использован при анимировании движения автомобиля, полета самолета или движения камеры. Траектория задается предварительно, со всеми положениями, повротами и изменениями масштаба в ключевые моменты времени. При запуске цикла симуляции состояние объекта пересчитывается в каждом кадре, с применением линейной интерполяции для положения и масштабирования и сферической линейной интерполяции для кватернионов вращения. Для этого используется внутренний метод slerp() класса osg::Quat.

OSG предоставляет класс osg::AnimationPath для описания изменяющейся во времени траектории. Для добаления в траекторию контрольных точек, коответсвующих определенным моментам времени используется метод этого класса insert(). Контрольная точка описывается классом osg::AnimationPath::ControlPoint, конструктор которого принимает в качестве параметров позицию, и, опционально, параметры поворота объекта и масштабирование. Например

```cpp
osg::ref_ptr<osg::AnimationPath> path = new osg::AnimationPath;
path->insert(t1, osg::AnimationPath::ControlPoint(pos1, rot1, scale1));
path->insert(t2, ...);
```

Здесь t1, t2 - моменты времени в секундах; rot1 - параметр поворота в момент времени t1, описываемый кватернионом osg::Quat.

Возможно управление зацикливанием анимации через метод setLoopMode(). По-умолчанию включен режим LOOP - анимация будет непрерывно повторятся. Другие возможные значения: NO_LOOPING - проигрывание анимации один раз и SWING - циклическое проигрывание движения в прямом и обратном направлениях.

После выполнения всех инициализаций мы присоединяем объект osg::AnimationPath к встроенному объекту osg::AnimationPathCallback, являющемуся производным от класса osg::NodeCallback.

## Пример анимации движения по траектории

Теперь заставим нашу цессну двигаться по окружности с центром в точке (0,0,0). Положение самолета на траектории будет расчитываться путем линейной интерполяции положения и ориентации между ключеными кадрами.

**main.h**
```cpp
#ifndef		MAIN_H
#define		MAIN_H

#include    <osg/AnimationPath>
#include    <osg/MatrixTransform>
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
osg::AnimationPath *createAnimationPath(double radius, double time)
{
    osg::ref_ptr<osg::AnimationPath> path = new osg::AnimationPath;
    path->setLoopMode(osg::AnimationPath::LOOP);

    unsigned int numSamples = 32;
    double delta_yaw = 2.0 * osg::PI / (static_cast<double>(numSamples) - 1.0);
    double delta_time = time / static_cast<double>(numSamples);

    for (unsigned int i = 0; i < numSamples; ++i)
    {
        double yaw = delta_yaw * i;
        osg::Vec3d pos(radius * sin(yaw), radius * cos(yaw), 0.0);
        osg::Quat rot(-yaw, osg::Z_AXIS);

        path->insert(delta_time * i, osg::AnimationPath::ControlPoint(pos, rot));
    }

    return path.release();
}

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
int main(int argc, char *argv[])
{
    (void) argc; (void) argv;

    osg::ref_ptr<osg::Node> model = osgDB::readNodeFile("../data/cessna.osg.0,0,90.rot");

    osg::ref_ptr<osg::MatrixTransform> root = new osg::MatrixTransform;
    root->addChild(model.get());

    osg::ref_ptr<osg::AnimationPathCallback> apcb = new osg::AnimationPathCallback;
    apcb->setAnimationPath(createAnimationPath(50.0, 6.0));

    root->setUpdateCallback(apcb.get());

    osgViewer::Viewer viewer;
    viewer.setSceneData(root.get());
    
    return viewer.run();
}
```

![](https://habrastorage.org/webt/ax/01/rl/ax01rlkfna5cfkvzqk3qfao_lmq.gif)
