# .proファイルの深層

ビルドボタンをクリックすると、Qt Creatorは何をしているのでしょうか？Qtは、単一の.proファイルで異なるプラットフォームのコンパイルをどのように扱うのでしょうか？Q_OBJECT マクロは正確には何を意味しているのでしょうか？次のセクションでは、これらの質問をそれぞれ掘り下げていきます。先ほど完成させた SysInfo アプリケーションを例に、Qt がその下で何をしているのかを勉強していきます。

この調査は.proファイルを掘り下げることから始めることができます。.pro ファイルは、Qt プロジェクトをコンパイルする際の主要なエントリーポイントです。基本的に、.pro ファイルはプロジェクトで使用されるソースとヘッダを記述した qmake プロジェクトファイルです。これはプラットフォームに依存しないMakefileの定義です。まず、ch02-sysinfoアプリケーションで使用されている異なるqmakeキーワードについて説明します。

```QMake
QT       += core gui charts

greaterThan(QT_MAJOR_VERSION, 4): QT += widgets

CONFIG += C++14

# You can make your code fail to compile if it uses deprecated APIs.
# In order to do so, uncomment the following line.
#DEFINES += QT_DISABLE_DEPRECATED_BEFORE=0x060000    # disables all the APIs deprecated before Qt 6.0.0
```

これらの関数にはそれぞれ特定の役割があります。

* #: これは、行にコメントするために必要な接頭辞です。
* QT: プロジェクトで使用されているQtモジュールのリストです。プラットフォーム固有のMakefileでは、各値にモジュールヘッダと対応するライブラリリンクが含まれています。
* CONFIG: プロジェクトの設定オプションのリストです。ここでは、Makefile で C++14 のサポートを設定します。

ch02-sysinfoアプリケーションでは、直感的なスコープの仕組みを利用したプラットフォーム固有のコンパイルルールの利用を開始しました。

```QMake
windows {
    SOURCES += sysinfowindowsimpl.cpp
    HEADERS += sysinfowindowsimpl.h
}
```

これをMakefileでやらなければならないとしたら、まともにやる前に毛が抜けてしまうかもしれません（ハゲていることは言い訳になりません）。この構文はシンプルでありながら強力なもので、条件文にも使われます。例えば、デバッグのみでいくつかのファイルをビルドしたいとしましょう。あなたは以下のように書いたでしょう。

```QMake
windows {
    SOURCES += SysInfoWindowsImpl.cpp
    HEADERS += SysInfoWindowsImpl.h
    debug {
        SOURCES += DebugClass.cpp
        HEADERS += DebugClass.h
    }
}
```

debug スコープを windows 内にネストすることは、if (windows && debug) と同等です。スコーピングの仕組みはさらに柔軟です。この構文では、OR ブール演算子条件を持つことができます。

```QMake
windows|unix {
    SOURCES += SysInfoWindowsAndLinux.cpp
}
```

他のif/else文を持っていても構いません。

```QMake
windows|unix {
    SOURCES += SysInfoWindowsAndLinux.cpp
} else:macx {
    SOURCES += SysInfoMacImpl.cpp
} else {
    SOURCES += UltimateGenericSources.cpp
}
```

このコードスニペットでは、+= 演算子を使用しています。qmake ツールには、変数の動作を変更するための幅広い演算子が用意されています。

* =: この演算子は、変数を値に設定します。SOURCES = SysInfoWindowsImpl.cpp という構文は、singleSysInfoWindowsImpl.cpp の値を SOURCES 変数に代入します。
* +=: この演算子は、値を値のリストに追加します。これは、HEADERS、SOURCES、CONFIG などでよく使われるものです。
* -=: この演算子は、リストから値を削除します。たとえば、共通セクションに DEFINE = DEBUG_FLAG 構文を追加し、プラットフォーム固有のスコープ (Windows リリースなど) で DEFINE -= DEBUG_FLAG 構文を使用してそれを削除することができます。
* \*=: この演算子は、値がまだ存在しない場合にのみリストに追加します。DEFINE \*= DEBUG_FLAG 構文は、DEBUG_FLAG 値を 1 回だけ追加します。
* ~=: この演算子は、正規表現に一致するすべての値を指定された値で置き換えます (DEFINE ~= s/DEBUG_FLAG/debug)。

.proファイルで変数を定義し、異なる場所で再利用することもできます。qmake message() 関数を使用することで、これを簡単にすることができます。

```QMake
COMPILE_MSG = "Compiling on"
windows {
    SOURCES += SysInfoWindowsImpl.cpp
    HEADERS += SysInfoWindowsImpl.h
    message($$COMPILE_MSG windows)
}
linux {
    SOURCES += SysInfoLinuxImpl.cpp
    HEADERS += SysInfoLinuxImpl.h
    message($$COMPILE_MSG linux)
}
macx {
    SOURCES += SysInfoMacImpl.cpp
    HEADERS += SysInfoMacImpl.h
    message($$COMPILE_MSG mac)
}
```

プロジェクトをビルドすると、プロジェクトをビルドするたびにプラットフォーム固有のメッセージが 「一般メッセージ」 タブに表示されます (このタブは、ウィンドウ | 出力ペイン | 一般メッセージ からアクセスできます)。ここでは、COMPILE_MSG 変数を定義し、 message($$COMPILE_MSG windows) を呼び出す際にこれを参照しています。これは、.proファイルから外部ライブラリをコンパイルする必要がある場合に興味深い可能性を提供します。変数内のすべてのソースを集約したり、特定のコンパイラへの呼び出しと組み合わせたりすることができます。

## Tip

スコープ固有の文が1行の場合は、以下の構文で記述できます。

windows:message($$COMPILE_MSG windows)


message() の他にも、いくつかの便利な関数があります。

* error(string): この関数は文字列を表示し、直ちにコンパイルを終了します。
* exists(filename): この関数は、filenameの存在をテストします。qmake は !演算子も提供していますので、 !exist(myfile) { ... } と書くこともできます。}.
* include(filename): この関数は、別の .pro ファイルの内容をインクルードします。これにより、.proファイルをより多くのモジュール化されたコンポーネントに分割することができます。これは、1つの大きなプロジェクトに複数の.proファイルがある場合に非常に便利です。

すべての機能は <http://doc.qt.io/qt-5/qmake-test-function-reference.html> に記載されています。

***
**[戻る](../index.htmnl)**
