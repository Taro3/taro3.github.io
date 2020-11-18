# 細かいことはいいから、とにかく ListView(QML) を使いたい

環境: Linux Mint 20 + Qt 5.15.1

**[全ソースはここ](https://github.com/Taro3/QListWidgetSample)**

ってことで、 ListView(QML) の使い方です。

シンプルなサンプルにしようと思ったんですが、結構ごちゃごちゃしてます。^^;
しかも、 QML にはそんなに詳しくないので尚更であります。
QML ってなんかダラ〜っと長い感じの実装になりがちでどうも…。
C++ から使えればいいのになぁ…と思う今日この頃。

## 表示するデータ(モデル)を用意する

ListViewに表示するデータを準備します。
ListView 用のモデルは、 ListModel で定義します。

### 初期データの定義

モデルの定義と同時にデータを定義してみます。

```QML
    // ListModel を作る
    ListModel {
        id: lmodel

        ListElement {           // ListElement でモデルの要素を定義する
            name: "item1"
            iconIndex: 1
            bgColor: "white"
        }
        ListElement {
            name: "item2"
            iconIndex: 2
            bgColor: "red"
        }
        ListElement {
            name: "item3"
            iconIndex: 3
            bgColor: "blue"
        }
    }
```

ListElement を使って、データ３つのを定義しています。
例では、 name 、 iconIndex 、 bgColor という３つのプロパティーを持っているデータを定義しています。

### 動的にモデルにデータを追加する

QML 内でモデルに動的に要素を追加する場合は、 ListModel.append を使います。

```QML
                // append で要素を追加する
                lmodel.append({ "name": name, "iconIndex": iconIndex, "bgColor": bgColor });
```

### 要素を削除する

モデル内の要素を削除する場合は、 ListModel.remove を使います。

```QML
                    // remove で要素を削除する
                    lmodel.remove(list.currentIndex);
```

### モデルの要素を取得する

モデル内の指定インデックスの要素を取得するには、 ListModel.get を使います。

```QML
                    // get でモデル要素を取得する
                    var t = JSON.stringify(lmodel.get(i));
```

### モデルの内容をクリアする

モデルの要素をすべて削除するには、 ListModel.clear を使います。

```QML
            /// clear でモデル要素をすべて削除する
            onClearClicked: lmodel.clear()
```

## ビューの作成

### リストを表示する

リストを表示するには、 ListView を使います。

```QML
ListView {
    id: root
    focus: true

    ...
}
```

### リスト内の１要素の表示方法を指定する

ListView 内の、各要素をどのように表示するかは、 ListView.delegate で指定します。

```QML
    delegate: Rectangle {
        width: root.width
        height: 20
        // bgColor を使って背景色を設定
        color: bgColor
        radius: index == root.currentIndex ? 4 : 0
        border.color: "black"
        border.width: index == root.currentIndex ? 2 : 0

        // iconIndex を使ってアイコン描画
        Image {
            id: icon
            width: 20
            height: 20
            source: "qrc:///icons/" + iconIndex + ".png"
        }

        // name を使ってテキストを描画
        Text {
            anchors.left: icon.right
            anchors.right: parent.right
            anchors.bottom: parent.bottom
            text: name
            font.bold: index == root.currentIndex ? true : false
        }
    }
```

この例では、背景色( bgColor )とアイコンインデックス( iconIndex )とテキスト( name )をモデルから受け取って要素を描画しています。

## 要素を選択する

リスト内のアイテムをクリックして選択状態にしてみます。

```QML
    MouseArea {
        anchors.fill: root

        onClicked: {
            var idx = root.indexAt(mouseX, mouseY);
            root.currentIndex = idx;
            root.itemClicked(idx)
        }
    }
```

要素を選択状態にするには、 ListView.currentIndex に選択インデックスを指定する必要があります。
この例では、クリックを検出するためには、 MouseArea を配置して、 MouseArea.onClicked で、 currentIndex を設定しています。

## おまけ

ListView の内容をファイルに保存/読込してみます。
Settings というコンポーネントを使って、データを永続化する方法もあるのですが、基本的に Settings はアプリケーションの設定値などを保存するために使うので、データ保存には普通にファイル保存したいところです。
そこで、 C++ でデータの保存/読込処理を作って QML から呼び出すことにします。

```C++
void DataIO::saveData(QVariantList list)
{
    QFile file(DATA_FILE_NAME);
    if (!file.open(QIODevice::WriteOnly | QIODevice::Text)) {
        qDebug() << "file open failed.";
        return;
    }
    for (const auto& j : list)
        file.write(j.toString().toUtf8() + "\n");
    file.close();
}
```

C++ で DataIO というクラスを作り、 saveData というメンバ関数を定義します。
引数の list には QML 側で作成した JSON テキストの配列が渡されます。

次にこのデータの読込用に loadData という関数も作ります。

```C++
QVariantList DataIO::loadData()
{
    QVariantList r;

    QFile file(DATA_FILE_NAME);
    if (!file.open(QIODevice::ReadOnly | QIODevice::Text)) {
        qDebug() << "file open failed.";
        return r;
    }

    while (true) {
        QString j = file.readLine();
        if (j.isEmpty())
            break;
        r.append(j);
    }
    file.close();

    return r;
}
```

これは、逆に読み込んだ JSON テキストを配列にして返すだけです。

QML から、 C++ 側の関数を呼び出すためには、まず C++ のヘッダで関数を Q_INVOKABLE を付けて宣言する必要があります。

```C++
class DataIO : public QObject
{
    Q_OBJECT
public:
    explicit DataIO(QObject *parent = nullptr);
    Q_INVOKABLE void saveData(QVariantList list);   // QML から呼び出せるようにする
    Q_INVOKABLE QVariantList loadData();
signals:

private:
    static const QString DATA_FILE_NAME;
}
```

更に、 main 関数で、 qmlRegisterType を使って QML のインポート情報を定義します。

```C++
int main(int argc, char *argv[])
{
    QCoreApplication::setAttribute(Qt::AA_EnableHighDpiScaling);

    QGuiApplication app(argc, argv);

    QQuickStyle::setStyle("Fusion");
    qmlRegisterType<DataIO>("DataIO", 1, 0, "DataIO");  // QML のインポート情報を定義する

    QQmlApplicationEngine engine;
    const QUrl url(QStringLiteral("qrc:/main.qml"));
    QObject::connect(&engine, &QQmlApplicationEngine::objectCreated,
                     &app, [url](QObject *obj, const QUrl &objUrl) {
        if (!obj && url == objUrl)
            QCoreApplication::exit(-1);
    }, Qt::QueuedConnection);
    engine.load(url);

    return app.exec();
}
```

最後に、 QML にインポート宣言を書きます。

```QML
import DataIO 1.0
```

これで、 QML 内で、 DataIO コンポーネントが使えるようになりますので宣言しておきます。

```QML
    DataIO { id: dataIo }
```

メンバ関数を呼び出す場合は、以下のようにします。

```QML
                dataIo.saveData(lsdata);
                ...
```

```QML
    var d = dataIo.loadData()
```

このサンプル内では、 **QML からの C++ 関数呼び出し**、 **JavaScript ファイル( .js )内の関数使用**、 **QML からのリソース取得**を行っています。

※ ListView の要素の保存には、もっといい方法があるのかもしれません…。^^;

***

**[戻る](../Qt.md)**
