# テストの実行

TestJsonSerializerというテストケースを書きました。私たちのドラムマシンテストアプリケーションにはmain()関数が必要です。以下の3つの可能性を探っていきます。

* QTEST_MAIN()関数。
* シンプルなmain()関数を書く。
* 複数のテストクラスをサポートする独自の拡張 main() 関数を書く。

QTestモジュールは、QTEST_MAIN()という興味深いマクロを提供しています。このマクロは、アプリケーションの完全なmain()関数を生成します。この生成されたメソッドは、テストケースのすべてのテスト関数を実行します。これを使用するには、TestJsonSerializer.cpp ファイルの最後に次のスニペットを追加します。

```C++
QTEST_MAIN(TestJsonSerializer)
```

さらに、テストクラスを .cpp ファイルのみで宣言して実装する場合 (ヘッダファイルなし)、生成された moc ファイルを QTEST_MAIN マクロ の後にインクルードする必要があります。

```C++
QTEST_MAIN(TestJsonSerializer)
#include "testjsonserializer"
```

QTEST_MAIN()マクロを使用する場合は、既存のmain.cppを削除することを忘れないでください。そうしないと、2つのmain()関数を持つことになり、コンパイルエラーが発生します。

これで、ドラムマシンテストアプリケーションを実行して、アプリケーションの出力を見ることができます。以下のようなものが表示されるはずです。

```shell
    $ ./drum-machine-test
    ********* Start testing of TestJsonSerializer *********
    Config: Using QtTest library 5.7.0, Qt 5.7.0 (x86_64-little_endian-lp64
shared (dynamic) release build; by GCC 4.9.1 20140922 (Red Hat 4.9.1-10))
    PASS : TestJsonSerializer::initTestCase()
    PASS : TestJsonSerializer::saveDummy()
    PASS : TestJsonSerializer::loadDummy()
    PASS : TestJsonSerializer::cleanupTestCase()
    Totals: 4 passed, 0 failed, 0 skipped, 0 blacklisted, 1ms
    ********* Finished testing of TestJsonSerializer *********
 ```

テスト関数である saveDummy() と loadDummy() は宣言順に実行されます。両方ともPASSステータスで成功しています。生成されたテストアプリケーションはいくつかのオプションを処理します。一般的には、次のコマンドを実行することでヘルプメニューを表示することができます。

```shell
$ ./drum-machine-test -help
```

クールな機能を見てみましょう。名前のついた関数を1つだけ実行することができます。以下のコマンドは、saveDummyテスト関数のみを実行します。

```shell
$ ./drum-machine-test saveDummy
```

また、複数のテスト関数をスペースで区切って実行することもできます。

QTestアプリケーションには、以下のようなログ詳細オプションがあります。

* -silentはサイレントモードです。致命的なエラーとサマリーメッセージのみを表示します。
* -v1 は verbose を表します。テスト関数に入力された情報を表示します。
* 拡張冗長語には -v2 を使用します。QCOMPARE()とQVERIFY()をそれぞれ表示します。
* 冗長な信号の場合は -vs。発信された信号と接続されたスロットを表示します。

例えば、以下のコマンドでloadDummyの実行の詳細を表示することができます。

```shell
    $ ./drum-machine-test -v2 loadDummy
    ********* Start testing of TestJsonSerializer *********
    Config: Using QtTest library 5.7.0, Qt 5.7.0 (x86_64-little_endian-lp64
shared (dynamic) release build; by GCC 4.9.1 20140922 (Red Hat 4.9.1-10))
    INFO : TestJsonSerializer::initTestCase() entering
    PASS : TestJsonSerializer::initTestCase()
    INFO : TestJsonSerializer::loadDummy() entering
    INFO : TestJsonSerializer::loadDummy() QCOMPARE(dummy.myInt, 1)
    Loc: [../../ch12-drum-machine-test/drum-machine￾test/TestJsonSerializer.cpp(45)]
    INFO : TestJsonSerializer::loadDummy() QCOMPARE(dummy.myDouble, 5.2)
    Loc: [../../ch12-drum-machine-test/drum-machine￾test/TestJsonSerializer.cpp(46)]
    INFO : TestJsonSerializer::loadDummy() QCOMPARE(dummy.myString,
QString("hello"))
    Loc: [../../ch12-drum-machine-test/drum-machine-
test/TestJsonSerializer.cpp(47)]
    INFO : TestJsonSerializer::loadDummy() QCOMPARE(dummy.myBool, true)
    Loc: [../../ch12-drum-machine-test/drum-machinetest/TestJsonSerializer.cpp(48)]
    PASS : TestJsonSerializer::loadDummy()
    INFO : TestJsonSerializer::cleanupTestCase() entering
    PASS : TestJsonSerializer::cleanupTestCase()
    Totals: 3 passed, 0 failed, 0 skipped, 0 blacklisted, 1ms
    ********* Finished testing of TestJsonSerializer *********
```

もう一つの大きな特徴は、ログ出力形式です。様々な形式（.txt、.xml、.csvなど）でテストレポートファイルを作成することができます。構文は、カンマで区切られたファイル名とファイル形式が必要です。

```shell
$ ./drum-machine-test -o <filename>,<format>
```

次の例では、test-report.xmlという名前のXMLレポートを作成します。

```shell
$ ./drum-machine-test -o test-report.xml,xml
```

いくつかのログレベルはプレーンテキスト出力にのみ影響することに注意してください。さらに、CSV 形式は、この章で後述するテストマクロ QBENCHMARK でのみ使用できます。

生成されたテストアプリケーションをカスタマイズしたい場合は、main()関数を記述します。TestJsonSerializer.cppのQTEST_MAINマクロを削除します。そして、以下のようなmain.cppを作成します。

```C++
#include "TestJsonSerializer.h"

int main(int argc, char *argv[])
{
    TestJsonSerializer test;
    QStringList arguments = QCoreApplication::arguments();
    return QTest::qExec(&test, arguments);
}
```

この場合、静的関数であるQTest::qExec()を使用して、TestJsonSerializerテストを開始しています。QTest CLIのオプションを楽しむために、コマンドライン引数を指定することを忘れないでください。

テスト関数を別のテストクラスで書いていた場合、テストクラスごとに1つのアプリケーションを作成していたことになります。テストアプリケーションごとに1つのテストクラスを残しておけば、QTEST_MAINマクロを使ってメイン関数を生成することもできます。

すべてのテストクラスを処理するために、1つのテストアプリケーションだけを作成したい場合があります。この場合、同じアプリケーション内に複数のテストクラスがあるので、各テストクラスに対して複数のメイン関数を生成したくないので、QTEST_MAINマクロを使用することはできません。

ユニークなアプリケーションですべてのテストクラスを呼び出す簡単な方法を見てみましょう。

```C++
int main(int argc, char *argv[])
{
    int status = 0;
    TestFoo testFoo;
    TestBar testBar;
    status |= QTest::qExec(&testFoo);
    status |= QTest::qExec(&testBar);
    return status;
}
```

このシンプルなカスタムmain()関数では、TestFooとTestBarテストを実行しています。しかし、CLIオプションを失っています。実際、コマンドライン引数でQTest::qExec()関数を複数回実行すると、エラーや悪い動作になります。例えば、TestBarから特定のテスト関数を1つだけ実行したい場合。TestFooを実行してもテスト関数が見つからず、エラーメッセージが表示され、アプリケーションが停止してしまいます。

ここでは、ユニークなアプリケーションで複数のテストクラスを処理するための回避策を紹介します。テストアプリケーションに-selectという新しいCLIオプションを作成します。このオプションを使用すると、実行する特定のテストクラスを選択することができます。以下に構文例を示します。

```shell
$ ./drum-machine-test -select foo fooTestFunctiono
```

select オプションを使用する場合は、コマンドの最初にテストクラス名 (この例では foo) をつけなければなりません。そして、オプションで Qt Test オプションを追加することができます。この目的を達成するために、新しい select オプションを解析し、対応するテストクラスを実行するための拡張 main() 関数を作成します。

強化されたmain()関数を一緒に作ります。

```C++
QApplication app(argc, argv);
QStringList arguments = QCoreApplication::arguments();

map<QString, unique_ptr<QObject>> tests;
tests.emplace("jsonserializer",
    make_unique<TestJsonSerializer>());
tests.emplace("foo", make_unique<TestFoo>());
tests.emplace("bar", make_unique<TestBar>());
```

このQAアプリケーションは、後に他のGUIテストケースで必要になります。後で使用するために、コマンドライン引数を取得します。testsというstd::mapテンプレートにはテストクラスのスマートポインタが含まれており、QStringラベルがキーとして使用されています。map::emplace() 関数を使用していますが、これはソースをマップにコピーするのではなく、その場で作成します。map::insert() 関数を使用すると、スマート・ポインタの不正コピーによるエラーが発生します。std::map テンプレートと make_unique で使用できる別の構文は以下の通りです。

```C++
tests["bar"] = make_unique<TestBar>();
```

これでコマンドラインの引数を解析することができます。

```C++
if (arguments.size() >= 3 && arguments[1] == "-select") {
    QString testName = arguments[2];
    auto iter = tests.begin();
    while(iter != tests.end()) {
        if (iter->first != testName) {
            iter = tests.erase(iter);
        } else {
            ++iter;
        }
    }
    arguments.removeOne("-select");
    arguments.removeOne(testName);
}
```

select オプションが使用されている場合、このスニペットは 2 つの重要なタスクを実行します。

* マップ・テストから、テスト名と一致しないテスト・クラスを削除します。
* select オプションと testName 変数の引数を削除し、QTest::qExec() 関数にクリーンな引数を提供します。

これで、テストクラスを実行するための最後のステップを追加することができます。

```C++
int status = 0;
for(auto& test : tests) {
    status |= QTest::qExec(test.second.get(), arguments);
}

return status;
```

selectオプションを指定しないと、すべてのテストクラスが実行されます。テストクラス名を指定して-selectオプションを使用すると、このテストクラスのみが実行されます。

***

**[戻る](../index.html)**
