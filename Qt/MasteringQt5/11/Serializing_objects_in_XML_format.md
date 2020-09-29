# XML 形式のオブジェクトのシリアライズ

JSONのシリアライズはC++オブジェクトを直接表現したもので、Qtはすでに必要なものをすべて提供してくれています。しかし、C++オブジェクトのシリアライズはXML形式で様々な表現が可能です。そこで、XML <-> QVariantの変換を自分たちで書かなければなりません。そこで、以下のようなXML表現を使うことにしました。

```xml
<[name]> type="[type]">[data]</[name]>
```

例えば、soundId型は以下のようなXML表現を提供します。

```xml
<soundId type="int">2</soundId>
```

Serializerも継承したC++クラスXmlSerializerを作成します。まずはsave()関数から始めてみましょう。

```C++
#include <QXmlStreamWriter>
#include <QXmlStreamReader>

#include "Serializer.h"

class XmlSerializer : public Serializer
{
public:
    XmlSerializer();
    void save(const Serializable& serializable,
    const QString& filepath,
    const QString& rootName) override;
};
```

これで、XmlSerializer.cppのsave()の実装を見ることができます。

```C++
void XmlSerializer::save(const Serializable& serializable, const QString&
    filepath, const QString& rootName)
{
    QFile file(filepath);
    file.open(QFile::WriteOnly);
    QXmlStreamWriter stream(&file);
    stream.setAutoFormatting(true);
    stream.writeStartDocument();
    writeVariantToStream(rootName, serializable.toVariant(),
        stream);
    stream.writeEndDocument();
    file.close();
}
```

ファイルパス先を指定してQFileファイルを作成します。QFileに書き込むQXmlStreamWriterオブジェクトを構築します。デフォルトでは、ライターはコンパクトなXMLを生成します。QXmlStreamWriter::setAutoFormatting()関数を使用して、きれいなXMLを生成することができます。QXmlStreamWriter::writeStartDocument()関数はXMLのバージョンとエンコーディングを書き込みます。XMLストリームにQVariantを書き込むには、writeVariantToStream()関数を使用します。最後に、ドキュメントを終了してQFileを閉じます。すでに説明したように、XMLストリームへのQVariantの書き込みは、データをどのように表現したいかに依存します。そのため、変換関数を書かなければなりません。このようにwriteVariantToStream()でクラスを更新してください。

```C++
//XmlSerializer.h
private:
    void writeVariantToStream(const QString& nodeName,
        const QVariant& variant, QXmlStreamWriter& stream);

//XmlSerializer.cpp
void XmlSerializer::writeVariantToStream(const QString& nodeName,
    const QVariant& variant, QXmlStreamWriter& stream)
{
    stream.writeStartElement(nodeName);
    stream.writeAttribute("type", variant.typeName());

    switch (variant.type()) {
        case QMetaType::QVariantList:
            writeVariantListToStream(variant, stream);
            break;

        case QMetaType::QVariantMap:
            writeVariantMapToStream(variant, stream);
            break;

        default:
            writeVariantValueToStream(variant, stream);
            break;
    }

    stream.writeEndElement();
}
```

この writeVariantToStream() 関数は一般的なエントリーポイントです。これは、XMLストリームにQVariantを配置したいときに毎回呼び出されます。QVariantクラスはリスト、マップ、データのいずれかになります。そのため、QVariantがコンテナ（QVariantListやQVariantMap）である場合には、特定の処理を適用します。それ以外の場合はデータ値とみなされます。以下、この関数の手順を説明します。

1. writeStartElement()関数で新しいXML要素を開始します。nodeNameはXMLタグの作成に使用されます。例えば、\<soundId.
2. 現在の要素にtypeというXML属性を記述します。QVariantに格納されている型の名前を使います。例えば、\<soundId type="int "とします。
3. QVariantデータ型に応じて、XMLシリアライズ関数の1つを呼び出します。例えば、\<soundId type="int">2.
4. 最後に、現在のXML要素をwriteEndElement()で終了させます。
   * 最終的な結果は \<soundId type="int"\>2\</soundId
   * この関数では、これから作成する3つのヘルパー関数を呼び出します。一番簡単なのはwriteVariantValueToStream()です。でXmlSerializerクラスを更新してください。

```C++
//XmlSerializer.h
void writeVariantValueToStream(const QVariant& variant,
    QXmlStreamWriter& stream);

//XmlSerializer.cpp
void XmlSerializer::writeVariantValueToStream(
    const QVariant& variant, QXmlStreamWriter& stream)
{
    stream.writeCharacters(variant.toString());
}
```

QVariant が単純型の場合は、その QString 表現を取得します。そして、QXmlStreamWriter::writeCharacters() を使用して、この QString を XML ストリームに書き込みます。

2 番目のヘルパー関数は writeVariantListToStream() です。以下にその実装を示します。

```C++
//XmlSerializer.h
private:
    void writeVariantListToStream(const QVariant& variant,
    QXmlStreamWriter& stream);

//XmlSerializer.cpp
void XmlSerializer::writeVariantListToStream(
    const QVariant& variant, QXmlStreamWriter& stream)
{
    QVariantList list = variant.toList();

    for(const QVariant& element : list) {
        writeVariantToStream("item", element, stream);
    }
}
```

このステップでは、QVariantがQVariantListであることがすでにわかっています。QVariant::toList() を呼び出してリストを取得します。次に、リストの全要素を反復処理し、一般的なエントリ・ポイントである writeVariantToStream() を呼び出します。リストから要素を取得しているので、要素名がないことに注意してください。しかし、リスト項目のシリアライズではタグ名は重要ではないので、任意のラベルitemを挿入します。

最後の書き込みヘルパー関数は、writeVariantMapToStream()です。

```C++
//XmlSerializer.h
private:
    void writeVariantMapToStream(const QVariant& variant,
        QXmlStreamWriter& stream);

//XmlSerializer.cpp
void XmlSerializer::writeVariantMapToStream(
    const QVariant& variant, QXmlStreamWriter& stream)
{
    QVariantMap map = variant.toMap();
    QMapIterator<QString, QVariant> i(map);

    while (i.hasNext()) {
        i.next();
        writeVariantToStream(i.key(), i.value(), stream);
    }
}
```

QVariantはコンテナですが、今回はQVariantMapです。見つかった要素ごとにwriteVariantToStream()を呼び出しています。これはマップなので、タグ名が重要です。ノード名としてQMapIterator::key()のマップキーを使用しています。

セーブ部分は終わりました。読み込みの部分を実装することができます。そのアーキテクチャは、保存関数と同じ精神に従っています。まずはload()関数から始めてみましょう。

```C++
//XmlSerializer.h
public:
    void load(Serializable& serializable,
        const QString& filepath) override;

//XmlSerializer.cpp
void XmlSerializer::load(Serializable& serializable,
    const QString& filepath)
{
    QFile file(filepath);
    file.open(QFile::ReadOnly);
    QXmlStreamReader stream(&file);
    stream.readNextStartElement();
    serializable.fromVariant(readVariantFromStream(stream));
}
```

まず、ソースのファイルパスでQFileを作成します。そのQFileを使ってQXmlStreamReaderを構築します。QXmlStreamReader ::readNextStartElement() 関数は、XML ストリームの次の開始要素まで読み込みます。次に、読み取りヘルパー関数 readVariantFromStream() を使用して、XML ストリームから QVariant クラスを作成することができます。最後に、Serializable::fromVariant() を使用して serializable を埋めます。ヘルパー関数readVariantFromStream()を実装してみましょう。

```C++
//XmlSerializer.h
private:
    QVariant readVariantFromStream(QXmlStreamReader& stream);

//XmlSerializer.cpp
QVariant XmlSerializer::readVariantFromStream(QXmlStreamReader& stream)
{
    QXmlStreamAttributes attributes = stream.attributes();
    QString typeString = attributes.value("type").toString();

    QVariant variant;
    switch (QVariant::nameToType(
    typeString.toStdString().c_str())) {
    case QMetaType::QVariantList:
        variant = readVariantListFromStream(stream);
        break;

    case QMetaType::QVariantMap:
        variant = readVariantMapFromStream(stream);
        break;

    default:
        variant = readVariantValueFromStream(stream);
        break;
    }

    return variant;
}
```

この関数の役割は、QVariantを作成することです。まず、XML属性から「type」を取得します。私たちの場合、扱う属性は1つだけです。そして、型に応じて、3つのリードヘルパー関数のうちの1つを呼び出します。それでは、readVariantValueFromStream()関数を実装してみましょう。

```C++
//XmlSerializer.h
private:
    QVariant readVariantValueFromStream(QXmlStreamReader& stream);

//XmlSerializer.cpp
QVariant XmlSerializer::readVariantValueFromStream(
    QXmlStreamReader& stream)
{
    QXmlStreamAttributes attributes = stream.attributes();
    QString typeString = attributes.value("type").toString();
    QString dataString = stream.readElementText();

    QVariant variant(dataString);
    variant.convert(QVariant::nameToType(
    typeString.toStdString().c_str()));
    return variant;
}
```

この関数は、型に応じたデータを持つQVariantを作成します。先ほどの関数と同様に、XML属性から型を取得します。また、QXmlStreamReader::readElementText()関数を用いて、データをテキストとして読み込みます。このQStringデータを使ってQVariantクラスを作成します。この段階では、QVariantの型はQStringになっています。そこで、QVariant::convert()関数を使用して、QVariantを実数型(int , qlonglong など)に変換します。

2 番目の読み込みヘルパー関数は readVariantListFromStream() です。

```C++
//XmlSerializer.h
private:
    QVariant readVariantListFromStream(QXmlStreamReader& stream);

//XmlSerializer.cpp
QVariant XmlSerializer::readVariantListFromStream(QXmlStreamReader& stream)
{
    QVariantList list;
    while(stream.readNextStartElement()) {
        list.append(readVariantFromStream(stream));
    }
    return list;
}
```

ストリーム要素には配列が含まれていることがわかっています。そこで、この関数はQVariantListを作成して返します。QXmlStreamReader::readNextStartElement() 関数は、次の開始要素まで読み込み、現在の要素内に開始要素が見つかった場合は真を返します。各要素ごとにエントリポイント関数 readVariantFromStream() を呼び出します。最後に、QVariantListを返します。

最後のヘルパー関数は readVariantMapFromStream() です。以下のスニペットでファイルを更新します。

```C++
//XmlSerializer.h
private:
    QVariant readVariantMapFromStream(QXmlStreamReader& stream);

//XmlSerializer.cpp
QVariant XmlSerializer::readVariantMapFromStream(
    QXmlStreamReader& stream)
{
    QVariantMap map;
    while(stream.readNextStartElement()) {
        map.insert(stream.name().toString(),
            readVariantFromStream(stream));
    }
    return map;
}
```

この関数はreadVariantListFromStream()のような感じです。今回はQVariantMapを作成する必要があります。新規項目の挿入に使用するキーは要素名です。QXmlStreamReader::name()関数で名前を取得します。

XmlSerializer でシリアライズされた Track クラスは以下のようになります。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<track type="QVariantMap">
    <duration type="qlonglong">6205</duration>
    <soundEvents type="QVariantList">
        <item type="QVariantMap">
            <soundId type="int">0</soundId>
            <timestamp type="qlonglong">2689</timestamp>
        </item>
            <item type="QVariantMap">
            <soundId type="int">2</soundId>
            <timestamp type="qlonglong">2690</timestamp>
        </item>
        <item type="QVariantMap">
            <soundId type="int">2</soundId>
            <timestamp type="qlonglong">3067</timestamp>
        </item>
    </soundEvents>
</track>
```

***

**[戻る](../index.html)**
