---
layout: post
title:  "Введение в OpenSceneGraph: Анимация объектов сцены"
date:   2018-11-07 15:20:00 +0300
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


