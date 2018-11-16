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

- описывает грань, задаваемую списком индексов вершин, принадлежащих данной грани. Модель в целом будем описывать такой структурой

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