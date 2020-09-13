# プラグインを Qt Designer に公開する

これで FilterWidget クラスが完成し、使えるようになりました。あとはFilterWidgetをQt Designerのプラグインシステムに登録しなければなりません。このグルーコードはQDesignerCustomWidgetInterfaceの子クラスを使って作っています。

FilterPluginDesignerという名前のC++クラスを新規作成し、FilterPluginDesigner.hをこのように更新します。

```C++
#include <QtUiPlugin/QDesignerCustomWidgetInterface>
class FilterPluginDesigner : public QObject, public QDesignerCustomWidgetInterface
{
    Q_OBJECT
    Q_PLUGIN_METADATA(IID
        "org.masteringqt.imagefilter.FilterWidgetPluginInterface")
    Q_INTERFACES(QDesignerCustomWidgetInterface)

public:
    FilterPluginDesigner(QObject* parent = 0);
};
```

FilterPluginクラスは2つのクラスを継承しています。

* QObject クラスは、Qt の親システムに依存しています。
* QDesignerCustomWidgetInterface クラスを使用して FilterWidget の情報をプラグインシステムに適切に公開します。

QDesignerCustomWidgetInterfaceクラスには、2つの新しいマクロが導入されています。

* Q_PLUGIN_METADATA() マクロは、メタオブジェクトシステムへのフィルタの一意の名前を示すためにクラスをアノテーションします。
* Q_INTERFACES() マクロは、現在のクラスが実装しているインターフェイスをメタオブジェクトシステムに伝えます。

Qt Designer がプラグインを検出できるようになりました。プラグイン自体についての情報を提供する必要があります。FilterPluginDesigner.hを更新します。

```C++
class FilterPluginDesigner : public QObject, public QDesignerCustomWidgetInterface
{
    ...
    FilterPluginDesigner(QObject* parent = 0);

    QStringname() const override;
    QStringgroup() const override;
    QStringtoolTip() const override;
    QStringwhatsThis() const override;
    QStringincludeFile() const override;
    QIconicon() const override;
    boolisContainer() const override;
    QWidget* createWidget(QWidget* parent) override;
    boolisInitialized() const override;
    void initialize(QDesignerFormEditorInterface* core) override;

private:
    boolmInitialized;
};
```

見た目よりも圧倒的に少ないです。これらの関数の本体は通常一行で構成されています。ここに最も簡単な関数の実装を示します。

```C++
QStringFilterPluginDesigner::name() const
{
    return "FilterWidget";
}

QStringFilterPluginDesigner::group() const
{
    return "Mastering Qt5";
}

QStringFilterPluginDesigner::toolTip() const
{
    return "A filtered picture";
}

QStringFilterPluginDesigner::whatsThis() const
{
    return "The filter widget applies an image effect";
}

QIconFilterPluginDesigner::icon() const
{
    returnQIcon(":/icon.jpg");
}

boolFilterPluginDesigner::isContainer() const
{
    return false;
}
```

ご覧のように、これらの関数についてはあまり説明することはありません。これらの関数のほとんどは、単に QString 値を返し、Qt Designer UI の適切な場所に表示されます。ここでは、最も興味深いものだけを取り上げます。まずは includeFile() から始めましょう。

```C++
QStringFilterPluginDesigner::includeFile() const
{
    return "FilterWidget.h";
}
```

この関数は uic (**User Interface Compiler**) から呼び出され、.ui ファイルに対応するヘッダを生成します。createWidget()の続きです。

```C++
QWidget* FilterPluginDesigner::createWidget(QWidget* parent)
{
    return new FilterWidget(parent);
}
```

この関数は、Qt Designer と FilterWidget の橋渡しをします。.ui ファイルに FilterWidget クラスを追加すると、Qt Designer は createWidget() 関数を呼び出して FilterWidget クラスのインスタンスを作成し、その内容を表示します。また、FilterWidget がアタッチされる _parent_ 要素も提供します。

最後に initialize() で終わりにしましょう。

```C++
voidFilterPluginDesigner::initialize(QDesignerFormEditorInterface*)
{
    if (mInitialized)
        return;

    mInitialized = true;
}
```

この関数では何もしていません。しかし、QDesignerFormEditorInterface*パラメータは説明する価値があります。Qt Designer が提供するこのポインタは、関数を介して Qt Designer のいくつかのコンポーネントにアクセスすることができます。

* actionEditor(): この機能は、アクションエディタです（デザイナーの下部パネル)
* formWindowManager(): この機能は、新しいフォームウィンドウを作成するためのインターフェイスです。
* objectInspector(): この機能は、レイアウトを階層的に表現したものです（デザイナーの右上パネル）。
* propertyEditor(): この関数は、現在選択されているウィジェット（デザイナーの右下パネル）の編集可能なすべてのプロパティのリストです。
* topLevel(): この機能は、デザイナーのトップレベルのウィジェットです。

これらのパネルについては、第 1 章「Qt入門」で説明しました。ウィジェットプラグインがこれらの領域に介入する必要がある場合、この関数は Qt Designer の動作をカスタマイズするためのエントリーポイントとなります。

***

**[戻る](../index.html)**
