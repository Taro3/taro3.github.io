# ListViewでアルバムを表示する

このモバイルアプリケーションの最初のページを作ってみましょうgallery.qrcにAlbumListPage.qmlというファイルを作成します。ここにページヘッダーの実装があります。

```QML
import QtQuick 2.0
import QtQuick.Layouts 1.3

import QtQuick.Controls 2.0

Page {

    header: ToolBar {
        Label {
            Layout.fillWidth: true
            text: "Albums"
            font.pointSize: 30
        }
    }
    ...
}
```

Pageは、ヘッダーとフッターを持つコンテナコントロールです。このアプリケーションでは、header項目のみを使用します。headerプロパティにToolBarを割り当てます。このツールバーの高さはQtで処理され、ターゲットプラットフォームに応じて調整されます。最初の簡単な実装では、"Albums "というテキストを表示するLabelを置くだけです。

このページの_header_初期化後にListView要素を追加します。

```QML
ListView {
    id: albumList
    model: albumModel
    spacing: 5
    anchors.fill: parent

    delegate: Rectangle {
        width: parent.width
        height: 120
        color: "#d0d1d2"

        Text {
            text: name
            font.pointSize: 16
            color: "#000000"
            anchors.verticalCenter: parent.verticalCenter
        }
    }
}
```

Qt Quick ListViewはQt WidgetのQListViewに相当します。提供されたmodelの項目のリストを表示します。modelプロパティの値をalbumModelに設定しています。これは、setContextProperty()関数を使用しているので、QMLからアクセス可能なmain.cppファイルのC++モデルを参照しています。Qt Quickでは、行の表示方法を記述するデリゲートを用意する必要があります。この場合、行にはText項目でアルバム名のみが表示されます。AlbumModel モデルはそのロールリストを QML に公開しているので、QML でアルバム名にアクセスするのは簡単です。ここで、AlbumModel のオーバーライドされた roleNames() 関数について思い出してみましょう。

```C++
QHash<int, QByteArray> AlbumModel::roleNames() const
{
    QHash<int, QByteArray> roles;
    roles[Roles::IdRole] = "id";
    roles[Roles::NameRole] = "name";
    return roles;
}
```

そのため、Qt Quick からのデリゲートが name ロールを使用するたびに、正しいロール整数で AlbumModel 関数 data() を呼び出し、正しいアルバム名の文字列を返します。

マウスを処理するには、行をクリックしてデリゲートにMouseArea要素を追加します。

```QML
ListView {
    ...
    delegate: Rectangle {
        ...
        MouseArea {
            anchors.fill: parent
            onClicked: {
                albumList.currentIndex = index
                pictureModel.setAlbumId(id)
                pageStack.push("qrc:/qml/AlbumPage.qml",
                    { albumName: name, albumRowIndex: index })
            }
        }
    }
}
```

MouseAreaは目に見えないアイテムで、マウスイベントを処理するために目に見えるアイテムと一緒に使うことができます。これは、電話のタッチスクリーンでの単純なタッチにも適用されます。ここでは、MouseArea要素に親のRectangleの全領域を取るように指示しています。

この例では、clickedシグナルに対してのみタスクを実行しています。ListViewのcurrentIndexをindexで更新します。このindexはモデル内の項目のインデックスを含む特別なロールです。

ユーザーがクリックしたら、 pictureModel.setAlbumId(id)コールで選択されたアルバムを読み込むように pictureModel に指示します。QMLがどのようにC++のメソッドを呼び出すことができるのか、すぐに見てみましょう。

最後に、AlbumPageをpageStackプロパティにプッシュします。push()関数は、{key: value, ... }構文を使ってQMLプロパティのリストを設定することができます。各プロパティは、プッシュされる項目にコピーされます。ここでは、name と index は、AlbumPage の albumName と albumRowIndex プロパティにコピーされます。これは、プロパティの引数を持つQMLページのインスタンスを作成するためのシンプルで強力な方法です。

QMLのコードからは、特定のC++メソッドのみを呼び出すことができます。

* プロパティ(Q_PROPERTYを使用)
* パブリックスロット
* invokableとして装飾された関数(Q_INVOKABLEを使用)

今回はPictureModel::setAlbumId()をQ_INVOKABLEとして装飾しますので、PictureModel.hファイルを更新してください。

```C++
class GALLERYCORESHARED_EXPORT PictureModel : public QAbstractListModel
{
    Q_OBJECT
public:
    ...
    Q_INVOKABLE void setAlbumId(int albumId);
    ...
};
```

***

**[戻る](5/index.html)**
