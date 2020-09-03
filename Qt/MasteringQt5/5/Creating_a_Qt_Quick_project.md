# Qt Quickプロジェクトの作成

この章では、第4章「デスクトップUIの克服」で説明したのと同じプロジェクト構造に従います。親プロジェクト ch05-gallery-mobile.pro は、2つのサブプロジェクト、gallery-coreと新しいgallery-mobileをホストします。

Qt creatorでは、**ファイル** → **新規ファイル**または**プロジェクト** → **アプリケーション** → **Qt Quick Controls Application** → **選択**からQt Quickサブプロジェクトを作成することができます。

ウィザードでは、プロジェクトの作成をカスタマイズすることができます。

* 場所
  * プロジェクト名(gallery-mobile)と場所を選択してください。
* 詳細
  * ui.qmlファイルで選択解除
  * 選択解除 ネイティブスタイリングを有効にする
* キット
  * デスクトップキットを選択
  * 少なくとも1つのモバイルキットを選択
* 概要
  * ch05-gallerymobile.proのサブプロジェクトとしてgallery-mobileを追加してください。

なぜこれらのオプションを使ってプロジェクトを作ったのか、少し時間をかけて説明してみましょう。

まず分析すべきはアプリケーションテンプレートです。デフォルトでは、Qt Quickは基本的なQMLコンポーネント(Rectangle, Image, Textなど)しか提供していません。高度なコンポーネントは Qt Quick モジュールで処理されます。このプロジェクトでは、Qt Quick コントロール（ApplicationWindow、Button、TextField など）を使用します。そのため、**Qt Quick Controls アプリケーション**から始めることにしました。Qt Quick モジュールはいつでも後でインポートして使うことができることを覚えておいてください。

この章では、Qt Quick Designer は使用しません。結果として、.ui.qmlファイルは必要ありません。デザイナーに助けられたとしても、自分でQMLファイルを理解して書くのが良いでしょう。

このプロジェクトは主にモバイルプラットフォームを対象としているため、デスクトップの「ネイティブスタイリング」を無効にしています。また、「ネイティブスタイリング」を無効にすることで、Qtウィジェットモジュールへの依存度が高くなることもありません。

最後に、少なくとも2つのキットを選択します。最初の1つはデスクトップキットです。他のキットはターゲットとするモバイルプラットフォームです。私たちは通常、以下のような開発ワークフローを使用します。

* デスクトップでの高速イテレーション
* モバイルエミュレータ/シミュレータでの動作確認と修正
* モバイル端末での実際のテスト

実機へのデプロイは一般的に時間がかかるので、ほとんどの開発はデスクトップキットで行うことができます。モバイルキットを使用すると、実際のモバイルデバイスやエミュレータ上でアプリケーションの動作を確認することができます (例えば Qt Android x86 キットを使用した場合など)。

ウィザードで自動生成されるファイルの話をしましょう。以下がmain.cppファイルです。

```C++
#include <QGuiApplication>
#include <QQmlApplicationEngine>

int main(int argc, char *argv[])
{
    QGuiApplication app(argc, argv);

    QQmlApplicationEngine engine;
    engine.load(QUrl(QStringLiteral("qrc:/main.qml")));

    return app.exec();
}
```

ここでは、Qtウィジェットを使わないので、QGuiApplicationではなくQGuiApplicationを使っています。そして、QMLエンジンを作成し、qrc:/mail.qmlをロードします。qrc:/という接頭辞からお察しの通り、このQMLファイルはQtリソースファイルの中にあります。

qml.qrcファイルを開くと、main.qmlを見つけることができます。

```QML
import QtQuick 2.5
import QtQuick.Controls 1.4

ApplicationWindow {
    visible: true
    width: 640
    height: 480
    title: qsTr("Hello World")

    menuBar: MenuBar {
        Menu {
            title: qsTr("File")
            MenuItem {
                text: qsTr("&Open")
                onTriggered: console.log("Open action triggered");
            }
            MenuItem {
                text: qsTr("Exit")
                onTriggered: Qt.quit();
            }
        }
    }

    Label {
        text: qsTr("Hello World")
        anchors.centerIn: parent
    }
}
```

まず、ファイル内で使用されている型をインポートします。各インポートの最後にあるモジュールのバージョンに注目してください。QtQuickモジュールは基本的なQML要素(Rectangle, Imageなど)をインポートし、QtQuick.ControlsモジュールはQtQuick Controlsサブモジュールから高度なQML要素(ApplicationWindow, MenuBar, MenuItem, Labelなど)をインポートします。

次に、ApplicationWindow型のルート要素を定義します。これは、以下の項目を持つトップレベルのアプリケーションウィンドウを提供します。メニューバー、ツールバー、ステータスバーです。ApplicationWindowのプロパティであるvisible、width、height、titleはプリミティブ型です。構文はシンプルでわかりやすいです。

menuBarプロパティはもっと複雑です。このMenuBarプロパティは、Menuファイルで構成されており、それ自体は2つのMenuItemsで構成されています。Open と Exit です。MenuItem は、それがアクティブになるたびに triggered() シグナルを発します。この場合、MenuItemファイルはコンソールにメッセージを記録します。 exit MenuItem はアプリケーションを終了させます。

最後に、ApplicationWindow型のコンテンツ・エリアに「Hello World」を表示するLabelを追加します。アンカーを使ってアイテムを配置すると便利です。この例では、ラベルは親の ApplicationWindow 内で縦と横の中央に配置されています。

先に進む前に、このサンプルがデスクトップとモバイルで正しく動作することを確認してください。

***

**[戻る](../index.html)**
