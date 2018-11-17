---
layout: post
title:  "Введение в OpenSceneGraph: Пример реализации собственного плагина - пишем и отлаживаем плагин"
date:   2018-11-16 22:02:00 +0300
categories: jekyll update
---

Этот пример в какой-то степени обобщает те знания, что мы уже получили об OSG из предылущих уроков. При написании плагина нам предстоит

1. Выбрать структуру данных для сохранения информации о геометрии модели, считанной из файла модели
2. Прочитать и разобрать (распарсить) файл с данными модели
3. Правильно настроить геометрический объект osg::Drawable по данным, прочитанным из файла
4. Построить субграф сцены для загруженной модели

## Реализуем каркас плагина

Итак, по традиции, приведу исходный код плагина целиком

**main.h**
```cpp
#ifndef		MAIN_H
#define		MAIN_H

#include    <osg/Geometry>
#include    <osg/Geode>
#include    <osgDB/FileNameUtils>
#include    <osgDB/FileUtils>
#include    <osgDB/Registry>
#include    <osgUtil/SmoothingVisitor>

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
struct face_t
{
    std::vector<unsigned int> indices;
};

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
struct pmd_mesh_t
{
    osg::ref_ptr<osg::Vec3Array> vertices;
    osg::ref_ptr<osg::Vec3Array> normals;
    std::vector<face_t> faces;

    pmd_mesh_t()
        : vertices(new osg::Vec3Array)
        , normals(new osg::Vec3Array)
    {

    }

    osg::Vec3 calcFaceNormal(const face_t &face) const
    {
        osg::Vec3 v0 = (*vertices)[face.indices[0]];
        osg::Vec3 v1 = (*vertices)[face.indices[1]];
        osg::Vec3 v2 = (*vertices)[face.indices[2]];

        osg::Vec3 n = (v1 - v0) ^ (v2 - v0);

        return n * (1 / n.length());
    }
};

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
class ReaderWriterPMD : public osgDB::ReaderWriter
{
public:

    ReaderWriterPMD();

    virtual ReadResult readNode(const std::string &filename,
                                const osgDB::Options *options) const;

    virtual ReadResult readNode(std::istream &stream,
                                const osgDB::Options *options) const;

private:

    pmd_mesh_t parsePMD(std::istream &stream) const;

    std::vector<std::string> parseLine(const std::string &line) const;
};

#endif
```

**main.cpp**
```cpp
#include	"main.h"

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
ReaderWriterPMD::ReaderWriterPMD()
{
    supportsExtension("pmd", "PMD model file");
}

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
osgDB::ReaderWriter::ReadResult ReaderWriterPMD::readNode(
        const std::string &filename,
        const osgDB::Options *options) const
{
    std::string ext = osgDB::getLowerCaseFileExtension(filename);

    if (!acceptsExtension(ext))
        return ReadResult::FILE_NOT_HANDLED;

    std::string fileName = osgDB::findDataFile(filename, options);

    if (fileName.empty())
        return ReadResult::FILE_NOT_FOUND;

    std::ifstream stream(fileName.c_str(), std::ios::in);

    if (!stream)
        return ReadResult::ERROR_IN_READING_FILE;

    return readNode(stream, options);
}

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
osgDB::ReaderWriter::ReadResult ReaderWriterPMD::readNode(
        std::istream &stream,
        const osgDB::Options *options) const
{
    (void) options;

    pmd_mesh_t mesh = parsePMD(stream);

    osg::ref_ptr<osg::Geometry> geom = new osg::Geometry;
    geom->setVertexArray(mesh.vertices.get());

    for (size_t i = 0; i < mesh.faces.size(); ++i)
    {
        osg::ref_ptr<osg::DrawElementsUInt> polygon = new osg::DrawElementsUInt(osg::PrimitiveSet::POLYGON, 0);

        for (size_t j = 0; j < mesh.faces[i].indices.size(); ++j)
            polygon->push_back(mesh.faces[i].indices[j]);

        geom->addPrimitiveSet(polygon.get());
    }

    geom->setNormalArray(mesh.normals.get());
    geom->setNormalBinding(osg::Geometry::BIND_PER_PRIMITIVE_SET);

    osg::ref_ptr<osg::Geode> geode = new osg::Geode;
    geode->addDrawable(geom.get());

    return geode.release();
}

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
pmd_mesh_t ReaderWriterPMD::parsePMD(std::istream &stream) const
{
    pmd_mesh_t mesh;

    while (!stream.eof())
    {
        std::string line;
        std::getline(stream, line);
        std::vector<std::string> tokens = parseLine(line);

        if (tokens[0] == "vertex")
        {
            osg::Vec3 point;
            std::istringstream iss(tokens[1]);
            iss >> point.x() >> point.y() >> point.z();
            mesh.vertices->push_back(point);
        }

        if (tokens[0] == "face")
        {
            unsigned int idx = 0;
            std::istringstream iss(tokens[1]);
            face_t face;

            while (!iss.eof())
            {
                iss >> idx;
                face.indices.push_back(idx);
            }

            mesh.faces.push_back(face);
            mesh.normals->push_back(mesh.calcFaceNormal(face));
        }
    }

    return mesh;
}

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
std::string delete_symbol(const std::string &str, char symbol)
{
    std::string tmp = str;
    tmp.erase(std::remove(tmp.begin(), tmp.end(), symbol), tmp.end());
    return tmp;
}

//------------------------------------------------------------------------------
//
//------------------------------------------------------------------------------
std::vector<std::string> ReaderWriterPMD::parseLine(const std::string &line) const
{
    std::vector<std::string> tokens;
    std::string tmp = delete_symbol(line, '\r');

    size_t pos = 0;
    std::string token;

    while ( (pos = tmp.find(':')) != std::string::npos )
    {
       token = tmp.substr(0, pos);
       tmp.erase(0, pos + 1);

       if (!token.empty())
           tokens.push_back(token);
    }

    tokens.push_back(tmp);

    return tokens;
}

REGISTER_OSGPLUGIN( pmd, ReaderWriterPMD )
```

Для начала позаботимся от структурах для хранения данных геометрии

```cpp
struct face_t
{
    std::vector<unsigned int> indices;
};
```

-- описывает грань, задаваемую списком индексов вершин, принадлежащих данной грани. Модель в целом будем описывать такой структурой

```cpp
struct pmd_mesh_t
{
    osg::ref_ptr<osg::Vec3Array> vertices;
    osg::ref_ptr<osg::Vec3Array> normals;
    std::vector<face_t> faces;

    pmd_mesh_t()
        : vertices(new osg::Vec3Array)
        , normals(new osg::Vec3Array)
    {

    }

    osg::Vec3 calcFaceNormal(const face_t &face) const
    {
        osg::Vec3 v0 = (*vertices)[face.indices[0]];
        osg::Vec3 v1 = (*vertices)[face.indices[1]];
        osg::Vec3 v2 = (*vertices)[face.indices[2]];

        osg::Vec3 n = (v1 - v0) ^ (v2 - v0);

        return n * (1 / n.length());
    }
};
```

Структура состоит содержит переменные-члены для хранения данных: vertices -- для храниения массива вершин геометрического объекта; normals -- массив номалей к граням объекта; faces - список граней объекта. В конструкторе структуры сразу выполняется инициализация умных указателей

```cpp
pmd_mesh_t()
        : vertices(new osg::Vec3Array)
        , normals(new osg::Vec3Array)
{

}
```

Кроме того, структура содержит метод, позволяющий расчитать вектор-нормаль к грани calcFaceNormal() в качестве параметра принимающий структуру, описывающую грань. В детали реализации этого метода мы пока не будетм вдаваться, резберем их несколько позже.

Таким образом, мы определились со структурами, в которых будем хранить данные геометрии. Теперь напишем каркас нашего плагина, а именно реализуем класс-наследник osgDB::ReaderWriter

```cpp
class ReaderWriterPMD : public osgDB::ReaderWriter
{
public:

    ReaderWriterPMD();

    virtual ReadResult readNode(const std::string &filename,
                                const osgDB::Options *options) const;

    virtual ReadResult readNode(std::istream &stream,
                                const osgDB::Options *options) const;

private:

    pmd_mesh_t parsePMD(std::istream &stream) const;

    std::vector<std::string> parseLine(const std::string &line) const;
};
```

Как и рекомендуется в описании API к разработке плагинов, в данном классе переопределяем методы чтения данных из файла и преобразования их в субграф сцены. У метода readNode() делаем две перегрузки - одна принимает на вход имя файла, другая - стандартный поток ввода. Конструктор класса определяет расширения файлов, поддерживаемых плагином

```cpp
ReaderWriterPMD::ReaderWriterPMD()
{
    supportsExtension("pmd", "PMD model file");
}
```

Первая перегрузка метода readNode() анализирует корректность имени файла и пути к нему, связывает с файлом стандартный поток ввода и вызывает вторую перегрузку, выполняющую основную работу

```cpp
osgDB::ReaderWriter::ReadResult ReaderWriterPMD::readNode(
        const std::string &filename,
        const osgDB::Options *options) const
{
    // Получаем расширение из пути к файлу
    std::string ext = osgDB::getLowerCaseFileExtension(filename);

    // Проверяем, поддерживает ли плагин это расширение
    if (!acceptsExtension(ext))
        return ReadResult::FILE_NOT_HANDLED;

    // Проверяем, имеется ли данный файл на диске
    std::string fileName = osgDB::findDataFile(filename, options);

    if (fileName.empty())
        return ReadResult::FILE_NOT_FOUND;

    // Связваем поток ввода с файлом
    std::ifstream stream(fileName.c_str(), std::ios::in);

    if (!stream)
        return ReadResult::ERROR_IN_READING_FILE;

    // Вызываем основную рабочую перегрузку метода readNode()
    return readNode(stream, options);
}
```

Во второй перегрузке реализуем алгоритм формирования объекта для OSG

```cpp
osgDB::ReaderWriter::ReadResult ReaderWriterPMD::readNode(
        std::istream &stream,
        const osgDB::Options *options) const
{
    (void) options;

    // Парсим файл *.pmd извлекая из него данные о геометрии
    pmd_mesh_t mesh = parsePMD(stream);

    // Создаем геометрию объекта
    osg::ref_ptr<osg::Geometry> geom = new osg::Geometry;
    // Задаем массив вершин
    geom->setVertexArray(mesh.vertices.get());

    // Формируем грани объекта
    for (size_t i = 0; i < mesh.faces.size(); ++i)
    {
        // Создаем примитив типа GL_POLYGON с пустым списком индексов вершин (второй параметр - 0)
        osg::ref_ptr<osg::DrawElementsUInt> polygon = new osg::DrawElementsUInt(osg::PrimitiveSet::POLYGON, 0);

        // Заполняем индексы вершин для текущей грани
        for (size_t j = 0; j < mesh.faces[i].indices.size(); ++j)
            polygon->push_back(mesh.faces[i].indices[j]);

        // Добаляем грань к геометрии
        geom->addPrimitiveSet(polygon.get());
    }

    // Задаем массив нормалей
    geom->setNormalArray(mesh.normals.get());
    // Указываем OpenGL, что каждая нормаль применяется к примитиву
    geom->setNormalBinding(osg::Geometry::BIND_PER_PRIMITIVE_SET);

    // Создаем листовой узел графа сцены и добавляем в него сформированную нами геометрию
    osg::ref_ptr<osg::Geode> geode = new osg::Geode;
    geode->addDrawable(geom.get());

    // Возвращаем готовый листовой узел
    return geode.release();
}
```

В конце файла main.cpp вызываем макрос REGISTER_OSGPLUGIN()

```cpp
REGISTER_OSGPLUGIN( pmd, ReaderWriterPMD )
```

Этот макрос формирует дополнительный код, позволяющий OSG, в лице бибилиотеки osgDB, сконструировать объект типа ReaderWriterPMD и вызвать его методы для загрузки файлов типа pmd. Таким образом, каркас плагин готов, дело осталось за малым -- реализовать загрузку и разбор файла pmd.

## Парсим файл 3D-модели

Теперь весь функционал плагина упирается в реализацию метода parsePMD()

```cpp
pmd_mesh_t ReaderWriterPMD::parsePMD(std::istream &stream) const
{
    pmd_mesh_t mesh;

    // Читаем файл построчно
    while (!stream.eof())
    {
        // Получаеми из файла очередную строку
        std::string line;
        std::getline(stream, line);

        // Разбиваем строку на состовлящие - тип данный и параметры
        std::vector<std::string> tokens = parseLine(line);

        // Если тип данных - вершина
        if (tokens[0] == "vertex")
        {
            // Читаем координаты вершины из списка параметров
            osg::Vec3 point;
            std::istringstream iss(tokens[1]);
            iss >> point.x() >> point.y() >> point.z();
            // Добавляем вершину в массив вершин
            mesh.vertices->push_back(point);
        }

        // Если тип данных - грань
        if (tokens[0] == "face")
        {
            // Читаем все индексы вершин грани из списка параметров
            unsigned int idx = 0;
            std::istringstream iss(tokens[1]);
            face_t face;

            while (!iss.eof())
            {
                iss >> idx;
                face.indices.push_back(idx);
            }

            // Добавляем грань в список граней
            mesh.faces.push_back(face);
            // Вычисляем нормаль к грани
            mesh.normals->push_back(mesh.calcFaceNormal(face));
        }
    }

    return mesh;
}
```

Метод parseLine() выполняет разбор строки pmd-файла

```cpp
std::vector<std::string> ReaderWriterPMD::parseLine(const std::string &line) const
{
    std::vector<std::string> tokens;
    // Формируем временную строку, удаляя из текущей строки символ возврата каретки (для Windows)
    std::string tmp = delete_symbol(line, '\r');

    size_t pos = 0;
    std::string token;

    // Ищем разделитель типа данных и параметров, разбивая строку на два токена:
    // тип данных и сами данные
    while ( (pos = tmp.find(':')) != std::string::npos )
    {
       // Выделяем токен типа данных (vertex или face в даннос случае) 
       token = tmp.substr(0, pos);
       // Удаляем найденный токен из строки вместе с разделителем
       tmp.erase(0, pos + 1);

       if (!token.empty())
           tokens.push_back(token);
    }

    // Помещаем оставшуюся часть строки в список токенов
    tokens.push_back(tmp);

    return tokens;
}
```

Этот метод предватит строку "vertex: 1.0 -1.0 0.0" в список двух строк "vertex" и " 1.0 -1.0 0.0". По первой строке мы идентифицируем тип данных - вершина или грань, из второй извлечем данные о координатах вершины. Для обеспечения работы этого метода нужна вспомогательная функция delete_symbol(), удаляющая из строки заданный символ и возвращающая строку не содержащую этого символа

```cpp
std::string delete_symbol(const std::string &str, char symbol)
{
    std::string tmp = str;
    tmp.erase(std::remove(tmp.begin(), tmp.end(), symbol), tmp.end());
    return tmp;
}
```

То есть теперь мы реализовали весь функционал нашего плагина и можем его протестировать.

## Тестируем плагин

Компилируем плагин и запускаем отладку (F5). Будет запущена отладочная версия стандартного просмотрщика osgviewerd, которая проанализиует переданный ей файл piramide.pmd, загрузит наш плагин и вызовет его метод readNode(). Если мы сделали всё правильно, то мы получим такой результат

![](https://habrastorage.org/webt/s3/5g/-y/s35g-ykmmlet5z8jisj7efkz7je.png)

Оказывается за списком вершин и граней в нашем придуманном фале 3D-модели скрывалась четырехугольная пирамида.

Зачем мы расчитывали нормали самостоятельно? В одном из уроков нам предлогался следующий метод автоматического расчета сглаженных нормалей

```cpp
osgUtil::SmoothingVisitor::smooth(*geom);
```

Применим эту функцию в нашем примере, вместо назначения собственных нормалей

```cpp
//geom->setNormalArray(mesh.normals.get());
//geom->setNormalBinding(osg::Geometry::BIND_PER_PRIMITIVE_SET);
osgUtil::SmoothingVisitor::smooth(*geom);
```

и мы получим следующий результат

![](https://habrastorage.org/webt/eb/gd/kt/ebgdktxsplxyqatprxunnwd8r2o.png)

Нормали влияют на расчет освещения модели, и мы видим что в данной ситуации сглаженные нормали приводят к некоорректным результатам расчета освещения пирамиды. Именно по этой причине мы пименили к расчету номалей свой велосипед. Но, думаю что объяснение нюансов этого выходит за рамки данного урока.