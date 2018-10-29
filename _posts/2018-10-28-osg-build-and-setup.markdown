---
layout: post
title:  "Введение в OpenSceneGraph: Сборка и установка"
date:   2018-10-28 09:19:00 +0300
categories: jekyll update
excerpt_separator:  <!--more-->
---

[OpenSceneGraph](http://www.openscenegraph.org/) - открытая кроссплатформенная библиотека, написанная на C++ и представляющая собой графический движок, предоставляющий программисту объектный интерфейс к библиотеке OpenGL. Несмотря на слабую известность в нашей стране, данный движок применяется за рубежем много где, например он является основой для свободного авиасимулятора FlightGear и ряда других проектов. Русскоязычной документации по нему исчезающе мало, а среди англоязычной можно отметить лишь серию книг от разработчиков: [OpenSceneGraph 3.0. Beginner's Guide](https://www.amazon.com/OpenSceneGraph-3-0-Beginners-Rui-Wang/dp/1849512825) и [OpenSceneGraph 3. Cookbook](https://www.amazon.com/OpenSceneGraph-3-Cookbook-Rui-Wang/dp/184951688X).

Моя личная заинтересованность в использовании этого движка привела к тому, что родилась эта серия статей. Их цель - дать вводные сведения для русскоязычных разработчиков, желающих использовать OpenSceneGraph (OSG) в своих проектах. В первой статье серии я расскажу о том, каким образом можно установить OSG на свой компьютер. Ориентироваться мы будем на операционную систему Windows, так как под линуксом всё и так достаточно просто.

<!--more-->

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

Для корректной работы cmake нам следует отредактировать файл **путь-установки-cmake\share\cmake-3.13\Modules\CMakeMinGWFindMake.cmake**. По-умолчанию он выгладит так

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

## Сборка и установка оладочной и релизной версий движка

Теперь запускаем командный интерпретатор cmd, ярлык на который находится по пути Пуск->Программы->Qt->Qt 5.11.2->Qt 5.11.2 for Desktop (MinGW 5.3.0 32bit)

![](https://habrastorage.org/webt/rp/wz/1n/rpwz1njpkzuicu-9iw2mabmzvbi.png)

Запущенный сеанс командной строки настраивает всё окружение, необходимое для работы средств сборки mingw32. Переходим в каталог с исходниками OSG
```
C:\Qt\Qt5.11.2\5.11.2\mingw53_32>D:
D:\> cd OSG\build-win32-debug
```

Даем команду

```
D:\OSG\build-win32-debug>cmake -G "MinGW Makefiles" -DCMAKE_INSTALL_PREFIX=E:\Apps\OSG -DCMAKE_BUILD_TYPE=DEBUG ../OpenSceneGraph
```

Разберем смысл параметров подробнее:
* -G "MinGW Makefiles" - указывает, что необходимо сгенерировать Makefile для утилиты mingw32-make 
* -DCMAKE_INSTALL_PREFIX=E:\Apps\OSG - устанавливаем путь, по которому будет установлен OSG
* -DCMAKE_BUILD_TYPE=DEBUG - указывает, что следует собирать отладочную версию движка.

Выполнение команды проверяет готовность окружения для сборки, генерирует сценарий сборки и следующий выхлоп

```
-- The C compiler identification is GNU 5.3.0
-- The CXX compiler identification is GNU 5.3.0
-- Check for working C compiler: C:/Qt/Qt5.11.2/Tools/mingw530_32/bin/gcc.exe
-- Check for working C compiler: C:/Qt/Qt5.11.2/Tools/mingw530_32/bin/gcc.exe -- works
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Detecting C compile features
-- Detecting C compile features - done
-- Check for working CXX compiler: C:/Qt/Qt5.11.2/Tools/mingw530_32/bin/g++.exe
-- Check for working CXX compiler: C:/Qt/Qt5.11.2/Tools/mingw530_32/bin/g++.exe -- works
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Looking for pthread.h
-- Looking for pthread.h - found
-- Looking for pthread_create
-- Looking for pthread_create - found
-- Found Threads: TRUE
-- Found OpenGL: opengl32
-- Could NOT find EGL (missing: EGL_INCLUDE_DIR)
-- Checking windows version...
-- Performing Test GL_HEADER_HAS_GLINT64
-- Performing Test GL_HEADER_HAS_GLINT64 - Failed
-- Performing Test GL_HEADER_HAS_GLUINT64
-- Performing Test GL_HEADER_HAS_GLUINT64 - Failed
-- 32 bit architecture detected
-- Could NOT find Freetype (missing: FREETYPE_LIBRARY FREETYPE_INCLUDE_DIRS)
-- Could NOT find JPEG (missing: JPEG_LIBRARY JPEG_INCLUDE_DIR)
-- Could NOT find Jasper (missing: JASPER_LIBRARIES JASPER_INCLUDE_DIR JPEG_LIBRARIES)
-- Could NOT find LibXml2 (missing: LIBXML2_LIBRARY LIBXML2_INCLUDE_DIR)
-- Could NOT find ZLIB (missing: ZLIB_INCLUDE_DIR)
-- Could NOT find ZLIB (missing: ZLIB_INCLUDE_DIR)
-- Could NOT find GDAL (missing: GDAL_LIBRARY GDAL_INCLUDE_DIR)
-- Could NOT find PkgConfig (missing: PKG_CONFIG_EXECUTABLE)
-- Could NOT find CURL (missing: CURL_LIBRARY CURL_INCLUDE_DIR)
-- Trying to find DCMTK expecting DCMTKConfig.cmake
-- Trying to find DCMTK expecting DCMTKConfig.cmake - failed
-- Trying to find DCMTK relying on FindDCMTK.cmake
-- Please set DCMTK_DIR and re-run configure (missing: DCMTK_config_INCLUDE_DIR DCMTK_dcmdata_INCLUDE_DIR DCMTK_dcmimage_INCLUDE_DIR DCMTK_dcmimgle_INCLUDE_DIR DCMTK_dcmjpeg_INCLUDE_DIR DCMTK_dcmjpls_INCLUDE_DIR DCMTK_dcmnet_INCLUDE_DIR DCMTK_dcmpstat_INCLUDE_DIR DCMTK_dcmqrdb_INCLUDE_DIR DCMTK_dcmsign_INCLUDE_DIR DCMTK_dcmsr_INCLUDE_DIR DCMTK_dcmtls_INCLUDE_DIR DCMTK_ofstd_INCLUDE_DIR DCMTK_oflog_INCLUDE_DIR)
-- Could NOT find PkgConfig (missing: PKG_CONFIG_EXECUTABLE)
-- Could NOT find GStreamer (missing: GSTREAMER_INCLUDE_DIRS GSTREAMER_LIBRARIES GSTREAMER_VERSION GSTREAMER_BASE_INCLUDE_DIRS GSTREAMER_BASE_LIBRARIES GSTREAMER_APP_INCLUDE_DIRS GSTREAMER_APP_LIBRARIES GSTREAMER_PBUTILS_INCLUDE_DIRS GSTREAMER_PBUTILS_LIBRARIES) (found version "")
-- Could NOT find SDL2 (missing: SDL2_LIBRARY SDL2_INCLUDE_DIR)
-- Could NOT find SDL (missing: SDL_LIBRARY SDL_INCLUDE_DIR)
-- Could NOT find PkgConfig (missing: PKG_CONFIG_EXECUTABLE)
-- Could NOT find PkgConfig (missing: PKG_CONFIG_EXECUTABLE)
-- Could NOT find PkgConfig (missing: PKG_CONFIG_EXECUTABLE)
-- Could NOT find JPEG (missing: JPEG_LIBRARY JPEG_INCLUDE_DIR)
-- Could NOT find ZLIB (missing: ZLIB_INCLUDE_DIR)
-- Could NOT find PNG (missing: PNG_LIBRARY PNG_PNG_INCLUDE_DIR)
-- Could NOT find TIFF (missing: TIFF_LIBRARY TIFF_INCLUDE_DIR)
-- g++ version 5.3.0
-- Performing Test _OPENTHREADS_ATOMIC_USE_GCC_BUILTINS
-- Performing Test _OPENTHREADS_ATOMIC_USE_GCC_BUILTINS - Success
-- Performing Test _OPENTHREADS_ATOMIC_USE_MIPOSPRO_BUILTINS
-- Performing Test _OPENTHREADS_ATOMIC_USE_MIPOSPRO_BUILTINS - Failed
-- Performing Test _OPENTHREADS_ATOMIC_USE_SUN
-- Performing Test _OPENTHREADS_ATOMIC_USE_SUN - Failed
-- Performing Test _OPENTHREADS_ATOMIC_USE_WIN32_INTERLOCKED
-- Performing Test _OPENTHREADS_ATOMIC_USE_WIN32_INTERLOCKED - Success
-- Performing Test _OPENTHREADS_ATOMIC_USE_BSD_ATOMIC
-- Performing Test _OPENTHREADS_ATOMIC_USE_BSD_ATOMIC - Failed
-- Configuring done
-- Generating done
-- Build files have been written to: D:/OSG/build-win32-debug
```

говорит нам о том, что можно приступать к сборке. Даем команду

```
D:\OSG\build-win32-debug>mingw32-make -j9
```
можно, как в моем примере, указать число потоков сборки, если у вас многоядерный процессор (ключ -j). Начнется процесс сборки, занимающий на моем компьютере около восьми минут

![](https://habrastorage.org/webt/ly/ng/_7/lyng_7y5k6a8zqm8207-3xeuufg.png)

По окончании сборки устанавливаем библиотеку
```
D:\OSG\build-win32-debug> mingw32-make install
```

после выполнения команды обнаруживаем библиотеку установленной по заранее заданному нами пути

![](https://habrastorage.org/webt/vl/no/n8/vlnon8cxexvn6lfab93h5haucu8.png)

Теперь соберем релизную версию движка, создав другой каталог сборки
```
D:\OSG\build-win32-debug>cd ..
D:\OSG> mkdir build-win32-release
D:\OSG>cd build-win32-release
D:\OSG\build-win32-release> cmake -G "MinGW Makefiles" -DCMAKE_INSTALL_PREFIX=E:\Apps\OSG ../OpenSceneGraph
D:\OSG\build-win32-release> mingw32-make -j9
D:\OSG\build-win32-release> mingw32-make install
```

## Настройка переменных окружения

Расположение библиотек OSG после инсталляции может быть любым - определяется это пожеланиями конкретного пользователя и его возможностями для размещения файлов на компьютере. При этом, при настройке конкретного проекта, использующего данные библиотеке требует стремится к некой унификации, абстрагируясь от конкретного расположения библиотек.

Создадим несколько системных переменных окружения, указывающих пути к библиотекам, заголовочным файлам и плагинам OSG. В приведенном мной примере это будет выглядеть так

![](https://habrastorage.org/webt/ef/gn/y1/efgny1zlztetepyi5wazz5o2pms.png)

Необходимо создать переменные, имена которых обведены на скриншоте красным. После создания переменных, для того, чтобы они были видны средствами разработки, в частности QtCreator-ом нужно как минимум перелогинится в системе (выйти и зайти от имени текущего пользователя) или, возможно, перезагрузить систему (это же Windows!)

После этого процедуру установки OSG на наш компьютер можно считать оконченой.


