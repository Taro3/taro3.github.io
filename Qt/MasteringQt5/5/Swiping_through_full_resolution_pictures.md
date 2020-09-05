# フル解像度の写真をスワイプで表示

gallery-mobileで最後に実装しなければならないページは、フル解像度の画像ページです。第4章の「デスクトップUIの克服」では、前/次のボタンを使って画像をナビゲートしました。この章では、モバイルプラットフォームをターゲットにしています。そのため、ナビゲーションはタッチベースのジェスチャーで行う必要があります。

この新しいPicturePage.qmlファイルの実装です。

```QML
import QtQuick 2.0
import QtQuick.Layouts 1.3
import QtQuick.Controls 2.0
import "."

PageTheme {

    property string pictureName
    property int pictureIndex

    toolbarTitle: pictureName

    ListView {
        id: pictureListView
        model: pictureModel
        anchors.fill: parent
        spacing: 5
        orientation: Qt.Horizontal
        snapMode: ListView.SnapOneItem
        currentIndex: pictureIndex

        Component.onCompleted: {
            positionViewAtIndex(currentIndex,
            ListView.SnapPosition)
        }

        delegate: Rectangle {
            property int itemIndex: index
            property string itemName: name

            width: ListView.view.width == 0 ?
            parent.width : ListView.view.width
            height: pictureListView.height
            color: "transparent"

            Image {
                fillMode: Image.PreserveAspectFit
                cache: false
                width: parent.width
                height: parent.height
                source: "image://pictures/" + index + "/full"
            }
        }
    }
}
```

まず、pictureNameとpictureIndexの2つのプロパティを定義します。現在のpictureNameプロパティはツールバーのタイトルに表示され、pictureIndexはListViewの正しい現在のインデックスを初期化するために使用され、currentIndex: pictureIndex となります。

画像をスワイプで表示するには、ListViewを使います。ここでは、各項目（単純なImage要素）は、その親のフルサイズを取ります。コンポーネントがロードされると、たとえ currentIndex が正しく設定されていたとしても、ビューは正しいインデックスに配置されるように更新されなければなりません。pictureListViewではこれを使っています。

```QML
Component.onCompleted: {
 positionViewAtIndex(currentIndex, ListView.SnapPosition)
}
```

これにより、現在表示されている項目の位置がcurrentIndexに更新されます。ここまでは順調です。それにもかかわらず、ListViewが作成されたとき、最初に行うことはデリゲートを初期化することです。ListViewはviewプロパティを持っていて、これはdelegateの内容で満たされています。つまり、ListView.view（はい、痛い）はComponent.onCompleted()では幅がないということになります。結果として、positionViewAtIndex()関数は何もしません。この動作を防ぐためには、デリゲートにデフォルトの初期幅を三項式 ListView.view.width == 0 ? parent.width : ListView.view.width で与えなければなりません。そうすると、ビューは最初のロード時にデフォルトの幅を持つことになり、 ListView.viewが適切にロードされるまでpositionViewAtIndex()関数は喜んで移動することができます。

各ピクチャをスワイプするために、ListViewのsnapModeの値をListView.SnapOneItemに設定しています。各スワイプは、動きを続けることなく、次の画像や前の画像にスナップします。

デリゲートのImage項目は、サムネイルバージョンと非常によく似ています。唯一の違いはソースプロパティで、ここではfull解像度のPictureImageProviderクラスを要求しています。

PicturePageを開くと、ヘッダーに正しいpictureNameプロパティが表示されます。しかし、ユーザーが別のピクチャに飛んでも名前は更新されません。これを処理するには、モーション状態を検出する必要があります。pictureListViewに以下のようなコールバックを追加します。

```QML
onMovementEnded: {
    currentIndex = itemAt(contentX, contentY).itemIndex
}

onCurrentItemChanged: {
    toolbarTitleLabel.text = currentItem.itemName
}
```

onMovementEnded()クラスは、スワイプで開始したモーションが終了したときにトリガされます。この関数では、ListViewcurrentIndexをcontentX, contentY座標で表示されている項目のitemIndexで更新しています。

2 番目の関数 onCurrentItemChanged() は、currentIndex の更新時に呼び出されます。これは単に現在のアイテムの画像名で toolbarTitleLabel.text を更新します。

PicturePage.qmlを表示するには、AlbumPage.qmlのthumbnailListデリゲートで同じMouseAreaパターンを使用します。

```QML
MouseArea {
    anchors.fill: parent
    onClicked: {
        thumbnailList.currentIndex = index
        pageStack.push("qrc:/qml/PicturePage.qml",
        { pictureName: name, pictureIndex: index })
    }
}
```

ここでも、PicturePage.qmlファイルがpageStackにプッシュされ、必要なパラメータ(pictureNameとpictureIndex)が同様に提供されます。

***

## まとめ

この章でギャラリーアプリケーションの開発を終了します。gallery-coreで強固な基盤を構築し、gallery-desktopでウィジェットUIを作成し、最後にgallery-mobileでQML UIを作成しました。

QMLは、UI開発への非常に迅速なアプローチを可能にします。残念ながら、この技術はまだ若く、急速に変化しています。モバイルOS(Android, iOS)との統合は激しい開発中であり、それがQtを使った素晴らしいモバイルアプリケーションにつながることを期待しています。今のところ、モバイルクロスプラットフォームツールキットの固有の限界を克服するのはまだ難しいです。

次の章では、Raspberry Pi上で動作するスネークゲームの開発という、QML技術を新たな境地へと導くことになります。

***

**[戻る](../index.html)**
