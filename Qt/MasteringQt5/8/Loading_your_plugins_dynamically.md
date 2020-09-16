# プラグインを動的にロードする

ここでは、これらのプラグインを読み込むアプリケーションの処理を行います。

1. ch08-image-animation内に新規**サブプロジェクト**を作成します。
2. **Qt ウィジェットアプリケーション**のタイプを選択します。
3. 名前を image-animation とし、デフォルトの**クラス情報の設定**を受け入れます。

.proファイルで最後にすべきことがいくつかあります。まず、イメージアニメーションは出力ディレクトリのどこかからプラグインを読み込もうとします。各フィルタプラグインプロジェクトは独立しているので、出力ディレクトリはイメージアニメーションとは分離されています。そのため、プラグインを修正するたびに、コンパイルされた共有ライブラリをイメージアニメーションの適切なディレクトリにコピーしなければなりません。これは image-animation アプリケーションで利用できるようにするために機能しますが、私たちは怠け者の開発者ですよね？

このようにplugins-common-priを更新することで自動化することができます。

```QMake
INCLUDEPATH += $$PWD/sdk
DEPENDPATH += $$PWD/sdk

windows {
    CONFIG(debug, debug|release) {
        target_install_path = $$OUT_PWD/../image-animation/debug/plugins/
    } else {
        target_install_path = $$OUT_PWD/../image-animation/release/plugins/
    }
} else {
    target_install_path = $$OUT_PWD/../image-animation/plugins/
}

# Check Qt file 'spec_post.prf' for more information about
'$$QMAKE_MKDIR_CMD'
createPluginsDir.path = $$target_install_path
createPluginsDir.commands = $$QMAKE_MKDIR_CMD $$createPluginsDir.path

INSTALLS += createPluginsDir

target.path = $$target_install_path
INSTALLS += target
```

一言で言えば、出力ライブラリは出力image-animation/pluginsディレクトリに配置されます。Windows では出力プロジェクトの構造が異なります。

さらに良いことに、createPluginsDir.commands = $$QMAKE_MKDIR_CMD $$createPluginsDir.path という命令で plugins ディレクトリが自動的に作成されます。システムコマンド (mkdir) の代わりに、特別な $$$QMAKE_MKDIR_CMD コマンドを使わなければなりません。Qtはこれを正しいシェルコマンド（OSによって異なります）に置き換えて、ディレクトリが既に存在しない場合にのみディレクトリを作成します。このタスクを実行するために、make installビルドステップを追加することを忘れないでください!

.pro ファイルで最後に行うべきことは、画像のアニメーションに関するものです。アプリケーションはFilterインスタンスを操作します。そのため、SDKにアクセスする必要があります。image-animation.proに以下を追加します。

```QMake
INCLUDEPATH += $$PWD/../sdk
DEPENDPATH += $$PWD/../sdk
```

シートベルトを締めてください。これから焼きたてのプラグインをロードしていきます。イメージアニメーションで、FilterLoaderという名前の新しいクラスを作成します。これが FilterLoader.h の内容です。

```C++
#include <memory>
#include <vector>
#include <Filter.h>

class FilterLoader
{
public:
    FilterLoader();
    void loadFilters();
    const std::vector<std::unique_ptr<Filter>>& filters() const;

private:
    std::vector<std::unique_ptr<Filter>> mFilters;
};
```

このクラスは、プラグインの読み込みを担当します。繰り返しになりますが、C++11 の unique_ptr を持つスマート ポインターに依存して、Filter インスタンスの所有権を明示しています。FilterLoader クラスは mFilters で所有者となり、filter() で vector へのゲッターを提供します。

filter() はベクトルに const& を返すことに注意してください。このセマンティックは二つの利点をもたらします。

* このリファレンスは vector がコピーされないようにしてくれます。これがなければ、コンパイラは「FilterLoaderはもうmFiltersのコンテンツの所有者ではありません！」などと吠えていたでしょう。もちろん、これは C++ テンプレートを扱うものなので、コンパイラのエラーはむしろ英語に対する驚異的な侮辱のように見えたでしょう。
* キーワード const は、vector 型が呼び出し元によって変更されないようにしています。

これで、FilterLoader.cpp: ファイルを作成することができます。

```C++
#include "FilterLoader.h"

#include <QApplication>
#include <QDir>
#include <QPluginLoader>

FilterLoader::FilterLoader() :
    mFilters()
{
}

void FilterLoader::loadFilters()
{
    QDir pluginsDir(QApplication::applicationDirPath());
#ifdef Q_OS_MAC
    pluginsDir.cdUp();
    pluginsDir.cdUp();
    pluginsDir.cdUp();
#endif
    pluginsDir.cd("plugins");

    for(QString fileName: pluginsDir.entryList(QDir::Files)) {
        QPluginLoader pluginLoader(
            pluginsDir.absoluteFilePath(fileName));
        QObject* plugin = pluginLoader.instance();
        if (plugin) {
            mFilters.push_back(std::unique_ptr<Filter>(
                qobject_cast<Filter*>(plugin)
            ));
        }
    }
}

const std::vector<std::unique_ptr<Filter>>& FilterLoader::filters() const
{
    return mFilters;
}
```

このクラスの本質は loadFilter() にあります。まず、plugins ディレクトリを pluginsDir で image-animation の出力ディレクトリに移動します。Macプラットフォームでは特殊なケースを扱います。QApplication::applicationDirPath()は、生成されたアプリケーションのバンドル内のパスを返します。外に出る唯一の方法は、cdUp()命令を使って3回上に登ることです。

このディレクトリ内の各fileNameに対して、QPluginLoaderローダーをロードしてみます。QPluginLoaderはQtプラグインへのアクセスを提供します。QPluginLoaderは、Qtのプラグインへのアクセスを提供します。さらに、QPluginLoaderローダーには以下のようなメリットがあります。

* プラグインがホストアプリケーションと同じバージョンの Qt とリンクされていることを確認します。
* C 関数に依存するのではなく、 instance() を介して直接プラグインにアクセスできるようにすることで、プラグインの読み込みを簡素化します。

次に pluginLoader.instance() を使用してプラグインのロードを試みます。これは、プラグインのルートコンポーネントの読み込みを試みます。この例では、ルートコンポーネントは FilerOriginal, FilterGrayscale, FilterBlur のいずれかです。この関数は常に QObject* を返します。プラグインをロードできなかった場合は 0 を返します。これが、カスタムプラグインで QObject クラスを継承した理由です。

instance()の呼び出しは、暗黙のうちにプラグインのロードを試みます。これが実行されると、QPluginLoaderはpluginのメモリを処理しません。ここからは、qobject_cast()を使ってプラグインをFilter*にキャストします。

qobject_cast() 関数は、標準的な C++ の dynamic_cast() と似たような動作をします。違いは、**RTTI (ランタイム型情報)** を必要としないことです。

最後になりましたが、Filter*にキャストされたpluginはunique_ptrの中でラップされ、mFilters vectorに追加されます。

***

**[戻る](../index.html)**
