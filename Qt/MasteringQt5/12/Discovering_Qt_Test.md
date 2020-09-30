# Qtテストの理解

Qt フレームワークは、C++ でユニットテストを作成するための完全な API である Qt Test を提供します。テストはアプリケーションのコードを実行して検証を行います。通常、テストは変数と期待される値を比較します。変数が特定の値と一致しない場合、テストは失敗します。さらに詳しく知りたい場合は、コードをベンチマークして、コードに必要な時間/CPUの刻み/イベントを取得することができます。GUI上で何度も何度もクリックしてテストするのは、すぐに退屈になってしまいます。Qt Testでは、ウィジェット上でキーボード入力やマウスイベントをシミュレートして、ソフトウェアを完全にチェックすることができます。

今回のケースでは、drum-machine-testというユニットテストプログラムを作成したいと思います。このコンソールアプリケーションは、前の章で紹介した有名なドラムマシンのコードをチェックします。ch12-drum-machine-testというsubdirsプロジェクトを以下のようなトポロジーで作成します。

* drum-machine:
    * drum-machine.pro
* drum-machine-test:
    * drum-machine-test.pro
* ch12-drum-machine-test.pro
* drum-machine-src.pri

ドラムマシンプロジェクトとドラムマシンテストプロジェクトは同じソースコードを共有しています。そのため、共通のファイルはすべてプロジェクトのインクルードファイル: drum-machine-src.priに置かれています。以下は更新された drum-machine.pro です。

```QMake
QT += core gui multimedia widgets
CONFIG += c++14

TARGET = drum-machine
TEMPLATE = app

include(../drum-machine-src.pri)

SOURCES += main.cpp
```

ご覧のように、私たちはリファクタリングタスクを実行しているだけで、プロジェクトの drum-machine は drum-machine-test アプリケーションの影響を受けません。これで、次のような drum-machine-test.pro ファイルを作成することができます。

```QMake
QT += core gui multimedia widgets testlib
CONFIG += c++14 console
TARGET = drum-machine-test
TEMPLATE = app

include(../drum-machine-src.pri)

DRUM_MACHINE_PATH = ../drum-machine
INCLUDEPATH += $$DRUM_MACHINE_PATH
DEPENDPATH += $$DRUM_MACHINE_PATH

SOURCES += main.cpp
```

最初に気づくべきことは、testlibモジュールを有効にする必要があるということです。次に、コンソールアプリケーションを作成している場合でも、GUI上でテストを行いたいので、プライマリアプリケーションで使用されるモジュール(gui, multimedia, widgets)もここで必要になります。最後に、すべてのアプリケーションファイル(ソース、ヘッダ、フォーム、リソース)を含むプロジェクトインクルードファイルをインクルードします。ドラムマシン・テスト・アプリケーションには新しいソース・ファイルも含まれているので、INCLUDEPATHとDEPENDPATH変数をソース・ファイル・フォルダに正しく設定しなければなりません。

Qt Test は使いやすく、いくつかの簡単な前提条件に依存しています。

* テストケースは QObject クラス
* プライベートスロットはテスト関数
* テストケースには、いくつかのテスト関数を含めることができます。

以下の名前のプライベートスロットはテスト関数ではなく、テストの初期化とクリーンアップのために自動的に呼び出される特別な関数であることに注意してください。

* initTestCase(): この関数は最初のテスト関数の前に呼び出されます。
* init(): この関数は、各テスト関数の前に呼び出されます。
* cleanup(): この関数は、各テスト関数の後に呼び出されます。
* cleanupTestCase(): この関数は、最後のテスト関数の後に呼び出されます。

さてさて、ドラムマシンテストアプリケーションの最初のテストケースを書く準備ができました。ドラムマシンオブジェクトのシリアル化は重要な部分です。セーブ機能の修正が悪いと、ロード機能が簡単に壊れてしまいます。コンパイル時にはエラーは出ませんが、使えないアプリケーションになってしまう可能性があります。だからこそテストが重要なのです。まず、シリアライズ/デシリアライズ処理を検証することです。DummySerializableという新しいC++クラスを作成します。ここにヘッダーファイルがあります。

```C++
#include "Serializable.h"

class DummySerializable : public Serializable
{
public:
    DummySerializable();
    QVariant toVariant() const override;
    void fromVariant(const QVariant& variant) override;

    int myInt = 0;
    double myDouble = 0.0;
    QString myString = "";
    bool myBool = false;
};
```

これは、第11章「シリアライズを楽しむ」で作成した Serializable インターフェイスを実装したシンプルなクラスです。このクラスは、シリアライズプロセスの下層を検証するのに役立ちます。ご覧のように、このクラスには様々な型の変数が含まれており、シリアライズが完全に機能するようになっています。DummySerializable.cppというファイルを見てみましょう。

```C++
#include "DummySerializable.h"

DummySerializable::DummySerializable() :
    Serializable()
{
}

QVariant DummySerializable::toVariant() const
{
    QVariantMap map;
    map.insert("myInt", myInt);
    map.insert("myDouble", myDouble);
    map.insert("myString", myString);
    map.insert("myBool", myBool);
    return map;
}

void DummySerializable::fromVariant(const QVariant& variant)
{
    QVariantMap map = variant.toMap();
    myInt = map.value("myInt").toInt();
    myDouble = map.value("myDouble").toDouble();
    myString = map.value("myString").toString();
    myBool = map.value("myBool").toBool();
}
```

ここでは驚くことはありません。前の章ですでに実行したように、QVariantMapを使って操作を実行します。ダミークラスの準備が整いました。次のヘッダを持つ新しいC++クラスTestJsonSerializerを作成します。

```C++
#include <QtTest/QTest>

#include "JsonSerializer.h"

class TestJsonSerializer : public QObject
{
    Q_OBJECT
public:
    TestJsonSerializer(QObject* parent = nullptr);

private slots:
    void cleanup();
    void saveDummy();
    void loadDummy();

private:
    QString loadFileContent();

private:
    JsonSerializer mSerializer;
};
```

最初のテストケースです！このテストケースは、JsonSerializerクラスの検証を行います。saveDummy()とloadDummy()の2つのテスト関数を見ることができます。cleanup()スロットは、先ほど説明した特別なQt Testスロットで、各テスト関数の後に実行されます。これで、TestJsonSerializer.cppに実装を書くことができます。

```C++
#include "DummySerializable.h"

const QString FILENAME = "test.json";
const QString DUMMY_FILE_CONTENT = "{\n "myBool": true,\n "myDouble":
5.2,\n "myInt": 1,\n "myString": "hello"\n}\n";

TestJsonSerializer::TestJsonSerializer(QObject* parent) :
    QObject(parent),
    mSerializer()
{}
```

ここでは2つの定数が作成されます。

* FILENAME: これは、データの保存とロードのテストに使用されるファイル名です。
* DUMMY_FILE_CONTENT: これは、テスト関数の saveDummy() および loadDummy() で使用される参照ファイルの内容です。

テスト関数 saveDummy() を実装してみましょう。

```C++
void TestJsonSerializer::saveDummy()
{
    DummySerializable dummy;
    dummy.myInt = 1;
    dummy.myDouble = 5.2;
    dummy.myString = "hello";
    dummy.myBool = true;

    mSerializer.save(dummy, FILENAME);

    QString data = loadFileContent();
    QVERIFY(data == DUMMY_FILE_CONTENT);
}
```

最初のステップは、いくつかの固定値を持つDummySerializableクラスをインスタンス化することです。そこで、テストする関数 JsonSerializer::save() を呼び出して、test.json ファイル内のダミーオブジェクトをシリアライズします。次に、ヘルパー関数であるloadFileContent()を呼び出して、test.jsonファイルに含まれるテキストを取得します。最後に、Qt TestマクロのQVERIFY()を使用して、JSONシリアライザによって保存されたテキストがDUMMY_FILE_CONTENTの期待値と同じであるかどうかの検証を行います。もしdataが正しい値と等しければ、テスト関数は成功します。以下がログ出力です。

```
PASS : TestJsonSerializer::saveDummy()
```

データが期待値と異なる場合、テストは失敗し、コンソール・ログにエラーが表示されます。

```
FAIL! : TestJsonSerializer::saveDummy()
'data == DUMMY_FILE_CONTENT' returned FALSE. ()
Loc: [../../ch12-drum-machine-test/drum-machinetest/TestJsonSerializer.cpp(31)]
```

ヘルパー関数であるloadFileContent()を簡単に見てみましょう。

```C++
QString TestJsonSerializer::loadFileContent()
{
    QFile file(FILENAME);
    file.open(QFile::ReadOnly);
    QString content = file.readAll();
    file.close();
    return content;
}
```

ここでは大したことはありません。test.jsonというファイルを開き、すべてのテキストコンテンツを読み込んで、対応するQStringを返します。

マクロである QVERIFY() は、ブール値をチェックするのに最適ですが、データを期待値と比較したい場合には、Qt Test の方が優れたマクロを提供しています。テスト関数 loadDummy(): で QCOMPARE() を発見してみましょう。

```C++
void TestJsonSerializer::loadDummy()
{
    QFile file(FILENAME);
    file.open(QFile::WriteOnly | QIODevice::Text);
    QTextStream out(&file);
    out << DUMMY_FILE_CONTENT;
    file.close();

    DummySerializable dummy;
    mSerializer.load(dummy, FILENAME);

    QCOMPARE(dummy.myInt, 1);
    QCOMPARE(dummy.myDouble, 5.2);
    QCOMPARE(dummy.myString, QString("hello"));
    QCOMPARE(dummy.myBool, true);
}
```

最初の部分では、参照内容のあるtest.jsonファイルを作成します。次に空のDymmySerializableを作成し、Serializable::load()をテストする関数を呼び出します。最後にQt TestマクロのQCOMPARE()を使用します。構文はシンプルです。

```C++
QCOMPARE(actual_value, expected_value);
```

これでJSONから読み込まれたダミーの各フィールドをテストすることができるようになりました。テスト関数であるloadDummmy()は、すべてのQCOMPARE()の呼び出しが成功した場合にのみ成功します。QCOMPARE()でのエラーは、もっと詳細です。

```
FAIL! : TestJsonSerializer::loadDummy() Compared values are not the same
 Actual (dummy.myInt): 0
 Expected (1) : 1
Loc: [../../ch12-drum-machine-test/drum-machinetest/TestJsonSerializer.cpp(45)]
```

テスト関数が実行されるたびに、特別な cleanup() スロットが呼び出されます。TestJsonSerializable.cppというファイルを次のように更新してみましょう。

```C++
void TestJsonSerializer::cleanup()
{
    QFile(FILENAME).remove();
}
```

これは、各テスト関数の後にtest.jsonファイルを削除し、保存テストとロードテストが衝突しないようにするシンプルなセキュリティです。

***

**[戻る](../index.html)**
