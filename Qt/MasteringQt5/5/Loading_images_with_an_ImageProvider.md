# ImageProviderで画像を読み込む

それは今、私たちの新しい永続化されたアルバムのためのサムネイルを表示する時間です。これらのサムネイルは、何らかの方法でロードしなければなりません。私たちのアプリケーションは、モバイルデバイスを対象としているので、我々はサムネイルをロードしている間、UIスレッドをフリーズさせる余裕がありません。私たちは、CPUを占有するか、OSによって殺されるかのどちらかになるでしょう、どちらもgallery-mobileのための望ましい運命ではありません。Qtは画像の読み込みを処理するための便利なクラスを提供しています。QQuickImageProviderです。

QQuickImageProviderクラスは、QMLコードでQPixmapクラスを非同期にロードするためのインターフェイスを提供します。このクラスは自動的にスレッドを生成して QPixmap クラスをロードするので、関数 requestPixmap() を実装するだけで済みます。QQuickImageProviderはデフォルトで要求されたpixmapをキャッシュして、データソースにヒットしすぎないようにしています。

サムネイルは、与えられた Picture の fileUrl にアクセスできる PictureModel 要素から読み込まなければなりません。QQuickImageProvider の実装では、PicturelModel の行インデックス用の QPixmap クラスを取得する必要があります。 PictureImageProvider という名前の新しい C++ クラスを作成し、PictureImageProvider.h を以下のように修正します。

```C++
#include <QQuickImageProvider>

class PictureModel;

class PictureImageProvider : public QQuickImageProvider
{
public:

    PictureImageProvider(PictureModel* pictureModel);

    QPixmap requestPixmap(const QString& id, QSize* size,
        const QSize& requestedSize) override;

private:
    PictureModel* mPictureModel;
};
```

fileUrl を取得するためには、コンストラクタで PictureModel 要素へのポインタを指定する必要があります。requestPixmap()をオーバーライドすることで、パラメータリストに id パラメータを指定することができます（size と requestedSize は今のところ無視して大丈夫です）。このidパラメータは、画像を読み込む際にQMLコードで指定します。QMLで指定されたImageに対して、PictureImageProviderクラスは以下のように呼ばれます。

```QML
Image { source: "image://pictures/" + index }
```

分解してみましょう。

* image: これは画像のURLソースのスキームです。これは、Qt が画像を読み込むために画像プロバイダと連携するように指示します。
* pictures: 画像プロバイダの識別子です。main.cppのQmlEngineの初期化時に、PictureImageProviderクラスとこの識別子をリンクします。
* index: 画像のIDです。ここでは画像の行インデックスです。requestPixmap()のidパラメータに対応します。

画像を2つのモードで表示したいことはすでにわかっています。サムネイルとフル解像度です。どちらの場合も QQuickImageProvider クラスを使用します。この2つのモードの動作は非常に似ています。これらのモードは、PictureModel に fileUrl を要求し、読み込まれた QPixmap を返します。

ここにパターンがあります。この2つのモードをPictureImageProviderで簡単にカプセル化することができます。唯一知っておかなければならないのは、呼び出し元がサムネイルかフル解像度のQPixmapかを知ることです。これは、idパラメータをより明確にすることで簡単に実現できます。

2つの方法で呼び出せるようにrequestPixmap()関数を実装する予定です。

* images://pictures/\<index\>/full: この構文を使用して、完全な解像度の画像を取得します。
* images://pictures/\<index\>/thumbnail: この構文を使用して、画像のサムネイル版を取得します。

もしindexの値が0の場合、この2回の呼び出しでrequestPixmap()でIDを0/fullまたは0/thumbnailに設定します。PictureImageProvider.cppの実装を見てみましょう。

```C++
#include "PictureModel.h"

PictureImageProvider::PictureImageProvider(PictureModel* pictureModel) :
    QQuickImageProvider(QQuickImageProvider::Pixmap),
    mPictureModel(pictureModel)
{
}

QPixmap PictureImageProvider::requestPixmap(const QString& id, QSize*
    /*size*/, const QSize& /*requestedSize*/)
{
    QStringList query = id.split('/');
    if (!mPictureModel || query.size() < 2) {
        return QPixmap();
    }

    int row = query[0].toInt();
    QString pictureSize = query[1];

    QUrl fileUrl = mPictureModel->data(mPictureModel->index(row, 0),
    PictureModel::Roles::UrlRole).toUrl();
    return ?? // Patience, the mystery will be soon unraveled
}
```

QQuickImageProvider::Pixmapパラメータを指定してQQuickImageProviderのコンストラクタを呼び出し、QQuickImageProviderがrequestPixmap()を呼び出すように設定します。QQuickImageProvider のコンストラクタは、さまざまな画像タイプ（QImage、QPixmap、QSGTexture、QQuickImageResponse）をサポートしており、それぞれに固有の requestXXX() 関数を持っています。

requestPixmap() 関数では、この ID を / で区切ることから始めます。ここから、行の値と希望する pictureSize を取得します。mPictureModel::data()関数を適切なパラメータで呼び出すだけで、fileUrlが読み込まれます。第 4 章の「デスクトップ UI を克服する」で全く同じ呼び出しを使用しました。

これで、どの fileUrl を読み込むべきか、必要な次元は何かがわかりました。しかし、最後に処理しなければならないことがあります。データベース ID ではなく行を操作するので、異なるアルバムにある 2 つの異なる画像に対して同じリクエスト URL を使用することになります。PictureModel は指定された Album の画像のリストをロードすることを覚えておいてください。

状況をイメージしてみましょう。Holidaysというアルバムでは、最初の写真を読み込むためのリクエストURLはimages://pictures/0/thumbnailになります。これは別のアルバム「ペット」でも同じURLで、最初の画像を読み込むためにはimages://pictures/0/thumbnailが必要になります。先ほども述べたように、QQuickImageProviderは自動的にキャッシュを生成し、同じURLのためのrequestPixmap()への後続の呼び出しを回避します。このように、どのアルバムが選択されていても、常に同じ画像を提供することができます。

この制約により、PictureImageProviderのキャッシュを無効にし、独自のキャッシュをロールアウトすることを余儀なくされています。これは面白いことになっています。ここに可能性のある実装があります。

```C++
// In PictureImageProvider.h

#include <QQuickImageProvider>
#include <QCache>

...
public:
    static const QSize THUMBNAIL_SIZE;
    QPixmap requestPixmap(const QString& id, QSize* size, const QSize& requestedSize) override;

    QPixmap* pictureFromCache(const QString& filepath, const QString& pictureSize);

private:
    PictureModel* mPictureModel;
    QCache<QString, QPixmap> mPicturesCache;
};

// In PictureImageProvider.cpp
const QString PICTURE_SIZE_FULL = "full";
const QString PICTURE_SIZE_THUMBNAIL = "thumbnail";
const QSize PictureImageProvider::THUMBNAIL_SIZE = QSize(350, 350);

QPixmap PictureImageProvider::requestPixmap(const QString& id, QSize*
/*size*/, const QSize& /*requestedSize*/)
{
    ...
    return *pictureFromCache(fileUrl.toLocalFile(), pictureSize);
}

QPixmap* PictureImageProvider::pictureFromCache(const QString& filepath,
const QString& pictureSize)
{
    QString key = QStringList{ pictureSize, filepath }
        .join("-");

    QPixmap* cachePicture = nullptr;
    if (!mPicturesCache.contains(pictureSize)) {
        QPixmap originalPicture(filepath);
        if (pictureSize == PICTURE_SIZE_THUMBNAIL) {
            cachePicture = new QPixmap(originalPicture
                .scaled(THUMBNAIL_SIZE,
            Qt::KeepAspectRatio,
            Qt::SmoothTransformation));
        } else if (pictureSize == PICTURE_SIZE_FULL) {
            cachePicture = new QPixmap(originalPicture);
        }
        mPicturesCache.insert(key, cachePicture);
    } else {
        cachePicture = mPicturesCache[pictureSize];
    }

    return cachePicture;
}
```

この新しい pictureFromCache() 関数は，生成された QPixmap を mPicturesCache に保存し，適切な QPixmap を返すことを目的としています．mPicturesCache クラスは QCache に依存しています。このクラスでは、各エントリにコストを割り当てることができ、キー/値の方法でデータを保存することができます。このコストは、オブジェクトのメモリコストとほぼ一致します（デフォルトでは cost = 1）。QCache がインスタンス化されると、maxCost 値（デフォルトでは 100）で初期化されます。すべてのオブジェクトの合計のコストがmaxCostを超えると、QCacheはオブジェクトの削除を開始して新しいオブジェクトのためのスペースを確保し、最近アクセスの少ないオブジェクトから始めます。

pictureFromCache() 関数では、キャッシュから QPixmap を取得しようとする前に、まず fileUrl と pictureSize からなるキーを生成します。これが存在しない場合は，適切な QPixmap（必要に応じて THUMBNAIL_SIZE マクロにスケーリングされたもの）が生成され，キャッシュ内に格納されます．mPicturesCache クラスがこの QPixmap の所有者になります。

PictureImageProviderクラスを完成させる最後のステップは、QMLコンテキストで利用できるようにすることです。これは main.cpp で行います。

```C++
#include "AlbumModel.h"
#include "PictureModel.h"
#include "PictureImageProvider.h"

int main(int argc, char *argv[])
{
    QGuiApplication app(argc, argv);
    ...

    QQmlContext* context = engine.rootContext();
    context->setContextProperty("thumbnailSize", PictureImageProvider::THUMBNAIL_SIZE.width());
    context->setContextProperty("albumModel", &albumModel);
    context->setContextProperty("pictureModel", &pictureModel);

    engine.addImageProvider("pictures", new
        PictureImageProvider(&pictureModel));
    ...
}
```

PictureImageProviderクラスは、engine.addImageProvider()でQMLエンジンに追加されます。最初の引数には、QMLのプロバイダ識別子を指定します。エンジンは、渡されたPictureImageProviderの所有権を取得することに注意してください。最後に、engineにはthumbnailSizeパラメータも渡されており、QMLコードで指定したサイズでサムネイルが表示されるように制約してくれます。

***

**[戻る](../index.html)**
