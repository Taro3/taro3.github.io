# コマンドラインインターフェイスの生成

コマンドラインインターフェースは、アプリケーションを特定のオプションで起動するための素晴らしい方法になります。Qt フレームワークは QCommandLineParser クラスでオプションを定義する簡単な方法を提供します。短い (例えば -t) または長い (例えば --test) オプション名を与えることができます。アプリケーションのバージョンとヘルプメニューは自動的に生成されます。オプションが設定されているかどうかをコードで簡単に取得することができます。オプションは値を取ることができ、デフォルト値を定義することができます。

例えば、ログファイルを設定するためのCLIを作成します。3つのオプションを定義したいと思います。

* 設定されている場合、-debug コマンドはログファイルの書き込みを有効にします。
* -f または --file コマンドは、ログを書き込む場所を定義します。
* -l または --level \<level\> コマンドは、ログの最小レベルを指定します。

以下のスニペットを見てください。

```C++
QCoreApplication app(argc, argv);

QCoreApplication::setApplicationName("ch14-hat-tips");
QCoreApplication::setApplicationVersion("1.0.0");

QCommandLineParser parser;
parser.setApplicationDescription("CLI helper");
parser.addHelpOption();
parser.addVersionOption();

parser.addOptions({
    {"debug", "Enable the debug mode."},
    \{\{"f", "file"}, "Write the logs into <file>.", "logfile"},
    \{\{"l", "level"}, "Restrict the logs to level <level>. Default is 'fatal'.", "level", "fatal"},
});

parser.process(app);

qDebug() << "debug mode:" << parser.isSet("debug");
qDebug() << "file:" << parser.value("file");
qDebug() << "level:" << parser.value("level");
```

それぞれのステップについてお話しましょう。

1. 最初の部分では、QCoreApplicationの関数を使用してアプリケーション名とバージョンを設定します。この情報は --version オプションで使用されます。
2. QCommandLineParserクラスのインスタンスを作成します。そして、ヘルプ(-hまたは--help)とバージョン(-vまたは--version)オプションを自動的に追加するように指示します。
3. QCommandLineParser::addOptions()関数でオプションを追加します。
4. コマンドライン引数を処理するように QCommandLineParser クラスに要求します。
5. オプションを取得して使用します。

オプションを作成するためのパラメータは以下の通りです。

* optionName: このパラメータを使用することで、単一または複数の名前を使用することができます。
* description: このパラメータでは、オプションの説明がヘルプメニューに表示されます。
* valueName (Optional): これは、あなたのオプションが値の名前を期待している場合に表示されます。
* defaultValue (Optional): これは、オプションのデフォルト値を表示します。

QCommandLineParser::isSet()を使用して、オプションを取得して使用することができます。オプションが値を必要とする場合、QCommandLineParser::value()でそれを取得することができます。

生成されたヘルプメニューの表示です。

```shell
$ ./ch14-hat-tips --help
Usage: ./ch14-hat-tips [options]
Helper of the command-line interface

Options:
  -h, --help Displays this help.
  -v, --version Displays version information.
  --debug Enable the debug mode.
  -f, --file <logfile> Write the logs into <file>.
  -l, --level <level> Restrict the logs to level <level>. Default is
'fatal'.
```

最後に、以下のスニペットは、使用中のCLIを表示します。

```shell
$ ./ch14-hat-tips --debug -f log.txt --level info
debug mode: true
file: "log.txt"
level: "info"
```

***

**[戻る](../index.html)**
