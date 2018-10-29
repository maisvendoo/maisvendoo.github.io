---
layout: post
title:  "Введение в OpenSceneGraph: Сборка и установка"
date:   2018-10-28 09:19:00 +0300
categories: jekyll update
---

[OpenSceneGraph](http://www.openscenegraph.org/) - открытая кроссплатформенная библиотека, написанная на C++ и представляющая собой графический движок, предоставляющий программисту объектный интерфейс к библиотеке OpenGL. Несмотря на слабую известность в нашей стране, данный движок применяется за рубежем много где, например он является основой для свободного авиасимулятора FlightGear и ряда других проектов. Русскоязычной документации по нему исчезающе мало, а среди англоязычной можно отметить лишь серию книг от разработчиков: [OpenSceneGraph 3.0. Beginner's Guide](https://www.amazon.com/OpenSceneGraph-3-0-Beginners-Rui-Wang/dp/1849512825) и [OpenSceneGraph 3. Cookbook](https://www.amazon.com/OpenSceneGraph-3-Cookbook-Rui-Wang/dp/184951688X).

Моя личная заинтересованность в использовании этого движка привела к тому, что родилась эта серия статей. Их цель - дать вводные сведения для русскоязычных разработчиков, желающих использовать OpenSceneGraph (OSG) в своих проектах. В первой статье серии я расскажу о том, каким образом можно установить OSG на свой компьютер. Ориентироваться мы будем на операционную систему Windows, так как под линуксом всё и так достаточно просто.

Единственным верным способом получить самую свежую версию OSG на своей машине - собрать библиотеку из исходных текстов. Существующих бинарный инталлятор для Windows ориентируется на компиляр MS Visual C++. Мне же, для своих проектов необходимо использование компилятора GCC, вернее его варианта MinGW32, входящего в поставку средств разработки фреймфорка Qt. Таким образом нам понадобится:

1. Установленный и настроенный [фреймворк Qt](https://www.qt.io/) с компилятором MinGW32 версии 5.3 и IDE QtCreator
2. Клиент [Git для Windows](https://git-scm.com/download/win)
3. Утилита [сmake для Windows](https://cmake.org/download/)  

Предполагается, что читатель знаком с IDE QtCreator и системой сборки qmake, используемой в проектах Qt. Кроме того, предполагается, что читатель владеет основами использования системы контроля версий Git и имеет ненулевые навыки в программировании в принципе.

## Получение исходных текстов OpenSceneGraph

Создаем на своем жестком диске каталог, где будем производить сборку OSG, например по пути D:\OSG

![](https://habrastorage.org/webt/sg/mb/ei/sgmbeipbjg0a7rp8tjzru6yxesa.png)

Перейдем в этот каталог и стянем туда исходники с [официального репозитория OSG на Github](https://github.com/openscenegraph/OpenSceneGraph)

```
D:\OSG> git clone https://github.com/openscenegraph/OpenSceneGraph.git
```

Длительность процесса скачивания зависит от того, насколько широк ваш канал доступа в Интернет. Рано или поздно мы получим у себя локальную копию репозитория OSG. 

Скачав исходники создадим рядом каталог build-win32-debug

![](https://habrastorage.org/webt/ah/cs/ug/ahcsugq_vwdefv1f_azxmpyjcc8.png)

В этом каталоге мы будем осуществлять сборку отладочного комплекта OSG. Но прежде

## Настройка cmake

Для корректной работы cmake нам следует отредактировать файл <путь к cmake>\share\cmake-x.yy\Modules\CMakeMinGWFindMake.cmake. По-умолчанию он выгладит так

```cmake
# Distributed under the OSI-approved BSD 3-Clause License.  See accompanying
# file Copyright.txt or https://cmake.org/licensing for details.


find_program(CMAKE_MAKE_PROGRAM mingw32-make.exe PATHS
  "[HKEY_LOCAL_MACHINE\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Uninstall\\MinGW;InstallLocation]/bin"
  c:/MinGW/bin /MinGW/bin
  "[HKEY_CURRENT_USER\\Software\\CodeBlocks;Path]/MinGW/bin"
  )
find_program(CMAKE_SH sh.exe )
if(CMAKE_SH)
  message(FATAL_ERROR "sh.exe was found in your PATH, here:\n${CMAKE_SH}\nFor MinGW make to work correctly sh.exe must NOT be in your path.\nRun cmake from a shell that does not have sh.exe in your PATH.\nIf you want to use a UNIX shell, then use MSYS Makefiles.\n")
  set(CMAKE_MAKE_PROGRAM NOTFOUND)
endif()

mark_as_advanced(CMAKE_MAKE_PROGRAM CMAKE_SH)
```

Закоментируем в нем несколько строк, дабы утилита не пыталась искать в нашей системе юниксовый шелл, и не найдя, завершалась с ошибкой

```cmake
# Distributed under the OSI-approved BSD 3-Clause License.  See accompanying
# file Copyright.txt or https://cmake.org/licensing for details.


find_program(CMAKE_MAKE_PROGRAM mingw32-make.exe PATHS
  "[HKEY_LOCAL_MACHINE\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Uninstall\\MinGW;InstallLocation]/bin"
  c:/MinGW/bin /MinGW/bin
  "[HKEY_CURRENT_USER\\Software\\CodeBlocks;Path]/MinGW/bin"
  )
#find_program(CMAKE_SH sh.exe )
#if(CMAKE_SH)
#  message(FATAL_ERROR "sh.exe was found in your PATH, here:\n${CMAKE_SH}\nFor MinGW make to work correctly sh.exe must NOT be in your path.\nRun cmake from a shell that does not have sh.exe in your PATH.\nIf you want to use a UNIX shell, then use MSYS Makefiles.\n")
#  set(CMAKE_MAKE_PROGRAM NOTFOUND)
#endif()

mark_as_advanced(CMAKE_MAKE_PROGRAM CMAKE_SH)
```

