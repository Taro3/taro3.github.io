# Qt3Dコードのエントリーポイントの作成

青春時代にスネークゲームをやっていなかった人のために、ここで簡単にゲーム性を覚えておきましょう。

* 誰もいない場所を移動するヘビを操作します。
* このエリアは壁に囲まれています。
* ゲームエリア内にランダムでリンゴがスポーンします。
* ヘビがリンゴを食べるとヘビが成長してポイントを獲得します。その直後に、ゲームエリアにもう一つのリンゴがスポーンします。
* ヘビが壁や自分の体の一部に触れた場合、そのヘビを失う。

目標は、できるだけ多くのリンゴを食べて最高得点を出すことです。蛇が長くなればなるほど、壁や自分の尻尾を避けるのが難しくなります。ああ、と蛇は、それがリンゴを食べるたびに、より速く、より速くなります。ゲームのアーキテクチャは以下のようになります。

* 全てのゲームアイテムはQMLでQt3Dを使用して定義されます。
* すべてのゲームロジックはJavaScriptで行われ、QML要素と通信します。

ゲームエリアの上にカメラを配置することで、オリジナルのスネークゲームの2D感を維持しつつ、3Dモデルやシェーダーでスパイスを効かせていきます。

さてさて、この瞬間のために多くのページを費やして準備してきました。いよいよスネーク・プロジェクトを始める時が来ました。ch06-snakeという名前の新しい**Qtクイック・コントロール・アプリケーション**を作成します。プロジェクトの詳細で

1. **最小限必要なQtのバージョン**欄は**Qt 5.6**を選択してください。
2. **ui.qmlファイル**のチェックを外します。
3. **ネイティブなスタイリングを有効にする**のチェックを外す。 
4. 「**次へ**」をクリックして、以下のキットを選択します。
   * RaspberryPi 2
   * **デスクトップ**
5. 「**次へ**」→「**完了**」をクリックします。

Qt3Dモジュールを追加する必要があります。ch06-snake.proをこのように変更します。

```QMake
TEMPLATE = app

QT += qml quick 3dcore 3drender 3dquick 3dinput 3dextras
CONFIG += c++11

SOURCES += main.cpp

RESOURCES += \
    snake.qrc

HEADERS +=

target.files = ch06-snake
target.path = /home/pi
INSTALLS += target
```

Qt3Dが動作するための適切なOpenGLコンテキストを持つために、アプリケーションのエントリーポイントを準備しなければなりません。main.cppを開いて、以下のように更新します。

```C++
#include <QGuiApplication>
#include <QtGui/QOpenGLContext>
#include <QtQuick/QQuickView>
#include <QtQml/QQmlEngine>

int main(int argc, char *argv[])
{
    QGuiApplication app(argc, argv);

    qputenv("QT3D_GLSL100_WORKAROUND", "");

    QSurfaceFormat format;
    if (QOpenGLContext::openGLModuleType() ==
        QOpenGLContext::LibGL) {
        format.setVersion(3, 2);
        format.setProfile(QSurfaceFormat::CoreProfile);
    }
    format.setDepthBufferSize(24);
    format.setStencilBufferSize(8);

    QQuickView view;
    view.setFormat(format);
    view.setResizeMode(QQuickView::SizeRootObjectToView);
    QObject::connect(view.engine(), &QQmlEngine::quit,
        &app, &QGuiApplication::quit);
    view.setSource(QUrl("qrc:/main.qml"));
    view.show();

    return app.exec();
}
```

このアイデアは、OpenGLを適切に扱うためにQSurfaceFormatを設定し、それをカスタムのQQuickView viewに与えることです。このビューは、このフォーマットを使用して自分自身を描画します。

qputenv("QT3D_GLSL100_WORKAROUND","")命令は、Raspberry Piなどの組み込みLinuxデバイス上のQt3Dシェーダに関連する回避策です。これにより、一部の組み込みデバイスで必要とされるライトのためのGLSL 1.00のスニペットを別個に用意できるようになります。この回避策を使用しないと、黒い画面が表示され、Raspberry Pi上でプロジェクトを正常に実行できなくなります。

***

## Tip

Qt3d lightsの回避策の詳細はこちら：<https://codereview.qt-project.org/#/c/143136/>。

***

ここでは、Qt Quickを使ってビューを扱うことにしました。もう一つの方法は、QMainWindowを継承するC++クラスを作成し、それをQMLコンテンツの親にすることです。この方法は、多くの Qt3D のサンプルプロジェクトで見ることができます。どちらも有効で、うまくいきます。QMainWindowアプローチではコードを書く量が多くなりがちですが、C++のみで3Dシーンを作成することができます。

main.cpp ファイルの view が main.qml ファイルを読み込もうとしていることに注意してください。これは main.qml です。

```QML
import QtQuick 2.6
import QtQuick.Controls 1.4

Item {
    id: mainView

    property int score: 0
    readonly property alias window: mainView

    width: 1280; height: 768
    visible: true

    Keys.onEscapePressed: {
        Qt.quit()
    }

    Rectangle {
        id: hud

        color: "#31363b"
        anchors.left: parent.left
        anchors.right: parent.right
        anchors.top : parent.top
        height: 60

        Label {
            id: snakeSizeText
            anchors.centerIn: parent
            font.pointSize: 25
            color: "white"
            text: "Score: " + score
        }
    }
}
```

ここでは、画面上部にある **HUD** (**ヘッドアップディスプレイ**) を定義し、そこにスコア (食べたリンゴの数) を表示します。EscapeキーをQt.quit()シグナルにバインドしていることに注意してください。このシグナルはmain.cppでQGuiApplication::quit()シグナルに接続してアプリケーションを終了させています。

これで QML コンテキストが Qt3D コンテンツを歓迎する準備が整いました。main.qmlを以下のように変更してください。

```QML
import QtQuick 2.6
import QtQuick.Controls 1.4
import QtQuick.Scene3D 2.0

Item {
    ...

    Rectangle {
        id: hud
        ...
    }

    Scene3D {
        id: scene
        anchors.top: hud.bottom
        anchors.bottom: parent.bottom
        anchors.left: parent.left
        anchors.right: parent.right
        focus: true
        aspects: "input"
    }
}
```

Scene3D要素は、hudオブジェクトの下の利用可能なスペースをすべて取ります。キーボードイベントを傍受できるように、ウィンドウのフォーカスを取ります。また、Qt3D エンジンがグラフ走査でキーボードイベントを処理できるようにするために、入力面を有効にしています。

***

**[戻る](../index.html)**
