# ThumbnailProxyModelの作成

将来の AlbumWidget ビューでは、選択された Album に添付された画像のサムネイルがグリッド状に表示されます。第3章「プロジェクトの分割とコードのルール」では、画像の表示方法を問わないように gallery-core ライブラリを設計しました。Picture クラスには mUrl フィールドしかありません。

つまり、サムネイルの生成は gallery-core ではなく gallery-desktop で行わなければなりません。PictureModel クラスは既に Picture 情報の取得を担当しているので、サムネイルデータを使ってその動作を拡張できるのは素晴らしいことです。

これは QAbstractProxyModel クラスとそのサブクラスを使用することで Qt で可能になります。このクラスの目的は、ベースとなる QAbstractItemModel からのデータを処理（ソート、フィルタリング、データの追加など）し、元のモデルをプロキシしてビューに提示することです。データベースに例えると、テーブルへの投影として見ることができます。

QAbstractProxyModel クラスには 2 つのサブクラスがあります。

* QIdentityProxyModelサブクラスは、ソースモデルを何も変更せずにプロキシします（すべてのインデックスが一致します）。このクラスは、data()関数を変換したい場合に適しています。
* QSortFilterProxyModel サブクラスは、渡されたデータをソートしたりフィルタリングしたりする機能を持つソースモデルをプロキシします。

前者のQIdentityProxyModelは、こちらの要件に合致しています。あとは、サムネイル生成の内容でdata()関数を拡張するだけです。ThumbnailProxyModelという名前のクラスを新規作成します。以下がThumbnailProxyModel.hファイルです。

```C++
#include <QIdentityProxyModel>
#include <QHash>
#include <QPixmap>

class PictureModel;

class ThumbnailProxyModel : public QIdentityProxyModel
{
public:
    ThumbnailProxyModel(QObject* parent = nullptr);

    PictureModel* PictureModel();

    // QAbstractItemModel interface
public:
    QVariant data(const QModelIndex &index, int role) const override;

    // QAbstractProxyModel interface
public:
    void setSourceModel(QAbstractItemModel *sourceModel) override;

private:
    void generateThumbnails(const QModelIndex& startIndex, int count);
    void reloadThumbnail();

private:
    QHash<QString, QPixmap*> mThumbnails;
};
```

このクラスは QIdentityProxyModel を継承し、いくつかの関数をオーバーライドします。

* サムネイルデータを ThumbnailProxyModel のクライアントに提供するための data() 関数
* sourceModelがエミットするシグナルに登録するsetSourceModel()関数

残りのカスタム関数は以下の目的を持っています。

* pictureModel() は、sourceModel を PictureModel* にキャストするヘルパー関数です。
* generateThumbnails()関数は、指定された画像のセットに対するQPixmapサムネイルの生成を行います。
* reloadThumbnails() は、 generateThumbnails() を呼び出す前に保存されているサムネイルをクリアするヘルパー関数です。

ご想像の通り、mThumbnailsクラスは、filepathをキーに使用してQPixmap*のサムネイルを保存します。

ThumbnailProxyModel.cpp ファイルに切り替えて、一から構築していきます。
ここでは、generateThumbnails()に注目してみましょう。

```C++
const unsigned int THUMBNAIL_SIZE = 350;
...
void ThumbnailProxyModel::generateThumbnails(const QModelIndex &startIndex, int count)
{
    if (!startIndex.isValid()) {
        return;
    }

    const QAbstractItemModel* model = startIndex.model();
    int lastIndex = startIndex.row() + count;
    for (int row = startIndex.row(); row < lastIndex; row++) {
        QString filepath = model->data(model->index(row, 0),
                                       PictureModel::Roles::FilePathRole).toString();
        QPixmap pixmap(filepath);
        auto thumbnail = new QPixmap(pixmap.scaled(THUMBNAIL_SIZE, THUMBNAIL_SIZE,
                                                   Qt::KeepAspectRatio,
                                                   Qt::SmoothTransformation));
        mThumbnails.insert(filepath, thumbnail);
    }
}
```

この関数は、パラメータ（startIndexとcount）で指定された範囲のサムネイルを生成します。各画像について、model->data() を用いて元のモデルからファイルパスを取得し、ダウンサイズされた QPixmap を生成して mThumbnails QHash に挿入します。サムネイルのサイズは const THUMBNAIL_SIZE で任意に設定していることに注意しましょう。画像はこのサイズに縮小され、元の画像のアスペクト比が尊重されます。

アルバムがロードされるたびに、我々はmThumbnailsクラスの内容をクリアし、新しい画像をロードする必要があります。この作業は reloadThumbnails() 関数で行います。

```C++
void ThumbnailProxyModel::reloadThumbnail()
{
    qDeleteAll(mThumbnails);
    mThumbnails.clear();
    generateThumbnails(index(0, 0), rowCount());
}
```

この関数では、単純にmThumbnailsの内容をクリアして、すべてのサムネイルを生成すべきことを示すパラメータを指定してgenerateThumbnails()関数を呼び出しています。では、この2つの関数がどのような時に使われるのか、setSourceModel()で見てみましょう。

```C++
void ThumbnailProxyModel::setSourceModel(QAbstractItemModel *sourceModel)
{
    QIdentityProxyModel::setSourceModel(sourceModel);
    if (!sourceModel) {
        return;
    }

    connect(sourceModel, &QAbstractItemModel::modelReset,
            [this] {
        reloadThumbnail();
    });

    connect(sourceModel, &QAbstractItemModel::rowsInserted,
            [this] (const QModelIndex& parent, int first, int last) {
        generateThumbnails(index(first, 0), last - first + 1);
    });
}
```

setSourceModel()関数が呼び出されると、ThumbnailProxyModelクラスは、どちらのベースモデルをプロキシすべきかを知るように設定されます。この関数では、元のモデルからエミットされる2つのシグナルにラムダを登録します。

* modelResetシグナルは、与えられたアルバムに対して画像がロードされるべき時にトリガーされます。この場合、サムネイルを完全にリロードしなければなりません。

* rowsInsertedシグナルは、新しい写真が追加されるたびにトリガされます。この時点で、これらの新しい写真でmThumbnailsを更新するために、generateThumbnailsを呼び出す必要があります。

最後に、data()関数を取り上げなければなりません。

```C++
QVariant ThumbnailProxyModel::data(const QModelIndex &index, int role) const
{
    if (role != Qt::DecorationRole) {
        return QIdentityProxyModel::data(index, role);
    }

    QString filepath = sourceModel()->data(index, PictureModel::Roles::FilePathRole).toString();
    return *mThumbnails[filepath];
}
```

Qt::DecorationRole以外のロールについては、親クラスのdata()が呼び出されます。私たちの場合、これは元のモデルであるPictureModelからdata()関数をトリガーしています。その後、data()がサムネイルを返さなければならない場合、indexで参照されるピクチャのfilepathが取得され、mThumbnailsのQPixmapオブジェクトを返すために使用されます。幸いなことに、QPixmap は暗黙のうちに QVariant にキャストできるので、ここで特別なことをする必要はありません。

ThumbnailProxyModelクラスで最後に取り上げる関数は、pictureModel()関数です。

```C++
ThumbnailProxyModel::PictureModel *ThumbnailProxyModel::PictureModel() const
{
    return static_cast<PictureModel*>(sourceModel());
}
```

ThumbnailProxyModel と対話するクラスは、ピクチャの作成や削除のために PictureModel に固有のいくつかの関数を呼び出す必要があります。この関数は sourceModel のキャストを PictureModel* に一元化するためのヘルパーです。

余談ですが、アルバムの読み込み中（および generateThumbnails() への呼び出し中）の初期ボトルネックを避けるために、その場でサムネイルを生成しようとすることができました。しかし、data() は const 関数であり、ThumbnailProxyModel インスタンスを変更できないことを意味します。これは、data() 関数でサムネイルを生成して mThumbnails に保存する方法を排除しています。

ご覧の通り、QIdentityProxyModel、そしてより一般的には QAbstractProxyModel は既存のモデルを壊すことなく動作を追加できる貴重なツールです。我々の場合、PictureModel クラスが gallery-desktop ではなく gallery-core で定義されている限り、これはデザインによって強制されています。PictureModel を変更することは gallery-core を変更することを意味し、ライブラリの他のユーザのためにその動作が壊される可能性があります。このアプローチでは、きれいに分離された状態を保つことができます。

***
**[戻る](../index.html)**
