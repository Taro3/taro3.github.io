# QMLシングルトンを用いたアプリケーションのプログラミング

QMLアプリケーションのスタイリングやテーマ設定は、様々な方法で行うことができます。本章では、カスタムコンポーネントで使用するテーマデータを持つQMLシングルトンを宣言します。また、ツールバーとそのデフォルト項目（戻るボタンとページのタイトル）を扱うカスタムPageコンポーネントを作成します。

Style.qmlファイルを新規作成してください。

```QML
pragma Singleton
import QtQuick 2.0

QtObject {
    property color text: "#000000"
    property color windowBackground: "#eff0f1"
    property color toolbarBackground: "#eff0f1"
    property color pageBackground: "#fcfcfc"
    property color buttonBackground: "#d0d1d2"
    property color itemHighlight: "#3daee9"
}
```

テーマのプロパティのみを格納する QtObject コンポーネントを宣言します。
QtObjectは非ビジュアルなQMLコンポーネントです。

QMLでシングルトン型を宣言するには二つのステップが必要です。まず、pragma singletonを使用する必要があります。これは、コンポーネントの単一インスタンスの使用を示すものです。次に、これを登録する必要があります。これはC++で行うか、qmldirファイルを作成することで行うことができます。2つ目のステップを見てみましょう。qmldirというプレーンテキストファイルを新規作成します。

```QML
singleton Style 1.0 Style.qml
```

この行は、Style.qmlというファイルからバージョン1.0のStyleというQMLのsingleton型を宣言するものです。

これらのテーマのプロパティをカスタムコンポーネントで使う時が来ました。簡単な例を見てみましょう。ToolBarTheme.qmlという名前のQMLファイルを新規作成します。

```QML
import QtQuick 2.0
import QtQuick.Controls 2.0

import "."

ToolBar {
    background: Rectangle {
        color: Style.toolbarBackground
    }

}
```

このQMLオブジェクトは、カスタマイズされたToolBarを記述しています。ここでは、background要素は、我々の色を持つ単純なRectangleです。シングルトンのStyleとそのテーマ・プロパティには、Style.toolbarBackgroundを使って簡単にアクセスすることができます。

***

## Info

QML シングルトンは qmldir ファイルを読み込むために明示的なインポートが必要です。import "."はこのQtのバグを回避するためです。詳しくは <https://bugreports.qt.io/browse/QTBUG-34418> をご覧ください。

***

ページのツールバーとテーマに関連するすべてのコードを含むことを目的として、QMLファイルPageTheme.qmlを作成します。

```QML
import QtQuick 2.0

import QtQuick.Layouts 1.3
import Qt.labs.controls 1.0
import QtQuick.Controls 2.0
import "."

Page {
    property alias toolbarButtons: buttonsLoader.sourceComponent
    property alias toolbarTitle: titleLabel.text

    header: ToolBarTheme {
        RowLayout {
            anchors.fill: parent
            ToolButton {
                background: Image {
                    source: "qrc:/res/icons/back.svg"
                }
                onClicked: {
                    if (stackView.depth > 1) {
                        stackView.pop()
                    }
                }
            }

            Label {
                id: titleLabel
                Layout.fillWidth: true
                color: Style.text
                elide: Text.ElideRight
                font.pointSize: 30
            }

            Loader {
                Layout.alignment: Qt.AlignRight
                id: buttonsLoader
            }
        }
    }

    Rectangle {
        color: Style.pageBackground
        anchors.fill: parent
    }
}
```

このPageTheme要素は、ページのヘッダーをカスタマイズします。先に作成したToolBarThemeを使用しています。このツールバーには、1行にアイテムを水平に表示するためのRowLayout要素のみが含まれています。このレイアウトには3つの要素が含まれています。

* ToolButton: これは、gallery.qrcからの画像を表示し、必要に応じて現在のページをポップアップ表示する「戻る」ボタンです。
* Label: ページタイトルを表示する要素です。
* Loader: これは、ページがこの汎用ツールバーに特定の項目を動的に追加することを可能にする要素です。

Loader要素は、sourceComponentプロパティを所有しています。このアプリケーションでは、このプロパティは特定のボタンを追加するためにPageThemeページによって割り当てることができます。これらのボタンは、実行時にインスタンス化されます。

PageThemeページには、親にフィットするRectangle要素も含まれており、Style.pageBackgroundを使用してページの背景色を設定します。

これで Style.qml と PageTheme.qml ファイルの準備ができたので、AlbumListPage.qml ファイルを更新して使用することができます。

```QML
import QtQuick 2.6
import QtQuick.Controls 2.0
import "."

PageTheme {

    toolbarTitle: "Albums"

    ListView {
        id: albumList
        model: albumModel
        spacing: 5
        anchors.fill: parent

        delegate: Rectangle {
            width: parent.width
            height: 120
            color: Style.buttonBackground

            Text {
                text: name
                font.pointSize: 16
                color: Style.text
                anchors.verticalCenter: parent.verticalCenter
            }
            ...
        }
    }
}
```

今、AlbumListPageはPageTheme要素であるので、headerを直接操作しません。ツールバーに素敵な "アルバム" テキストを表示するために、プロパティ toolbarTitle を設定する必要があるだけです。また、Styleシングルトンのプロパティを使って、素敵な色を楽しむことができます。

テーマのプロパティを1つのファイルに集約することで、アプリケーションの見た目を簡単に変えることができます。プロジェクトのソースコードには、ダークなテーマも含まれています。

***

**[戻る](../index.html)**
