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

Начинаем с создания траектории самолета, вынося этот код в отдельную функцию

```cpp
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
```

В качестве параметров функция принимает радиус окружности, по которой движется самолет и время, за которое он совершит один оборот. Внутри функции создаем объект траектории и включаем режим циклического повторения анимации

```cpp
osg::ref_ptr<osg::AnimationPath> path = new osg::AnimationPath;
path->setLoopMode(osg::AnimationPath::LOOP);
```

Следующий код

```cpp
unsigned int numSamples = 32;
double delta_yaw = 2.0 * osg::PI / (static_cast<double>(numSamples) - 1.0);
double delta_time = time / static_cast<double>(numSamples);
```

вычмсляет параметры аппроксимации траектории. Мы разбиваем всю траекторию на numSamples прмолинейных участков, и вычисляем изменение угла поворота самолета вокруг вертикальной оси (рыскания) delta_yaw и изменение времени delta_time при переходет от участка к учаcтку. Теперь создаем необходимые контрольные точки

```cpp
for (unsigned int i = 0; i < numSamples; ++i)
{
    double yaw = delta_yaw * i;
    osg::Vec3d pos(radius * sin(yaw), radius * cos(yaw), 0.0);
    osg::Quat rot(-yaw, osg::Z_AXIS);

    path->insert(delta_time * i, osg::AnimationPath::ControlPoint(pos, rot));
}
```

В цикле перебираются все учатки траектории от первого до последнего. Каждая контрольная точка характеризуется углом рыскания

```cpp
double yaw = delta_yaw * i;
```

положением центра масс самолета в пространстве

```cpp
osg::Vec3d pos(radius * sin(yaw), radius * cos(yaw), 0.0);
```

Поророт самолета на требуемый угол рыскания (относительно вертикальной оси) задаем кватернионом

```cpp
osg::Quat rot(-yaw, osg::Z_AXIS);
```

а затем добавляем расчитанные параметры в список контрольных точек траектории

```cpp
path->insert(delta_time * i, osg::AnimationPath::ControlPoint(pos, rot));
```

В основной программа обращаем внимание на нюанс в указании имени файла модели самолета при загрузке

```cpp
osg::ref_ptr<osg::Node> model = osgDB::readNodeFile("../data/cessna.osg.0,0,90.rot");
```

-- к имени файла добавился некий суффикс ".0,0,90.rot". Механизм загрузки геометрии из файла, используемый в OSG позволяет таким образом указать начальное положение и оринтацию модели после загрузки. В данном случае мы хотим, чтобы загрузившись, модель была повернута на 90 градусов вокруг оси Z.

Далее создается корневой узел, являющийся узлом трансформации, и объект модели добавляется к нему в качестве дочернего узла

```cpp
osg::ref_ptr<osg::MatrixTransform> root = new osg::MatrixTransform;
root->addChild(model.get());
```

Теперь создаем обратный вызов анимации траектории, добавляя в него путь, создаваемый функцией createAnimationPath()

```cpp
osg::ref_ptr<osg::AnimationPathCallback> apcb = new osg::AnimationPathCallback;
apcb->setAnimationPath(createAnimationPath(50.0, 6.0));
```

Прикрепляем этот коллбэк к узлу трансформации

```cpp
root->setUpdateCallback(apcb.get());
```

Инициализация и запуск вьювера производится как обычно

```cpp
osgViewer::Viewer viewer;
viewer.setSceneData(root.get());

return viewer.run();
```

Получаем анимацию движения самолета

![](https://habrastorage.org/webt/ax/01/rl/ax01rlkfna5cfkvzqk3qfao_lmq.gif)


Подумайте, вам ничего не показалось странным в этом примере? Ранее, например в программе, когда выполнялся рендеринг в текстуру, вы изменяли матрицу трансформации явно, чтобы добится изменения положения модели в пространстве. Здесь же мы только создаем узел трансформации и в коде нигде не просиходит явного задания матрицы. 

Секрет в том, что эту работу выполняет специальный класс osg::AnimationPathCallback. В соотвествии с текущи положением объекта на траектории он вычисляет матрицу трансформации и автоматически применяет её к тому узлу трансформации, к которому он прикреплен, избавляя разработчика от кучи рутинных операций.

Тут следует отметить, что прикрепление osg::AnimationPathCallback к другим типам узлов, не только не даст эффекта, но и может привести к неопределенному поведению программы. Важно помнить, что данный обратный вызов воздействует исключительно на узлы трансформации.

## Программное управление анимацией

Класс osg::AnimationPathCallback предоставляет методы для управления анимацией в процессе выполнения программы

1. reset() -- сбос анимации и проигрывание её сначала.
2. setPause() -- постановка анимации на паузу. В качестве параметра принимает логическое значение
3. setTimeOffset() -- задает смещение во времени до начала анимации.
4. setTimeMultiplier() - задает временной множитель для ускорения/замедления анимации.

Например, для снятия анимации с паузы и сброса выполняем такой код

```cpp
apcb->setPause(false);
apcb->reset();
```

а для начала анимации с четвортой секунды после запуска программы с двухкратным ускорением, такой код

```cpp
apcb->setTimeOffset(4.0f);
apcb->setTimeMultiplier(2.0f);
```

