# アプリケーション内部のプラグインを使用する

プラグインが適切にロードされたので、アプリケーションの UI からアクセスできるようにしなければなりません。そのためには、第7章「頭の痛いことのないサードパーティライブラリ」の FilterWidget クラスからインスピレーションを得ようとしています（恥知らずな盗用）。

FilterWidget という名前の **Widget** テンプレートを使って、新しい Qt Designer **Form Class** を作成します。FilterWidget.uiファイルは、第7章「頭を痛めないサードパーティライブラリ」で完成したものと全く同じです。

このようにFilterWidget.hファイルを作成します。

```C++
#include <QWidget>
#include <QImage>

namespace Ui {
class FilterWidget;
}

class Filter;

class FilterWidget : public QWidget
{
    Q_OBJECT
public:
    explicit FilterWidget(Filter& filter, QWidget *parent = 0);
    ~FilterWidget();

    void process();

    void setSourcePicture(const QImage& sourcePicture);
    void setSourceThumbnail(const QImage& sourceThumbnail);
    void updateThumbnail();

    QString title() const;

signals:
    void pictureProcessed(const QImage& picture);

protected:
    void mousePressEvent(QMouseEvent*) override;

private:
    Ui::FilterWidget *ui;
    Filter& mFilter;

    QImage mDefaultSourcePicture;
    QImage mSourcePicture;
    QImage mSourceThumbnail;

    QImage mFilteredPicture;
    QImage mFilteredThumbnail;
};
```

全体的には、Qt Designer プラグインに関するすべてのものを削除し、コンストラクタへの参照で mFilter の値を渡すだけにしました。FilterWidget クラスは Filter の所有者ではなく、むしろそれを呼び出すのはクライアントです。Filter (プラグイン) のオーナーは FilterLoader であることを覚えておいてください。

もう一つの変更点は、新しい setThumbnail() 関数です。これは古い updateThumbnail() の代わりに呼ばれるべきです。新しい updateThumbnail() はサムネイル処理のみを行い、ソースサムネイルには触れません。この分割は、来るべきアニメーション部分の作業を準備するために行われます。サムネイルの更新は、アニメーションが終了した後にのみ行われます。

***

## Info

FilterWidget.cppについては、この章のソースコードを参照してください。

***

これで全てのローレイヤーが完成しました。次はMainWindowを埋める作業です。ここでも、第7章「頭を痛めないサードパーティ・ライブラリ」で説明したのと同じパターンを踏襲しています。MainWindow.ui との唯一の違いは、filterLayout が空であることです。明らかに、プラグインは動的にロードされるので、コンパイル時にプラグインの中に何も入れるものがありません。

MainWindow.h を取り上げてみましょう。

```C++
#include <QMainWindow>
#include <QImage>
#include <QVector>

#include "FilterLoader.h"

namespace Ui {
class MainWindow;
}

class FilterWidget;

class MainWindow : public QMainWindow
{
    Q_OBJECT

public:
    explicit MainWindow(QWidget *parent = 0);
    ~MainWindow();

    void loadPicture();

protected:
    void resizeEvent(QResizeEvent* event) override;

private slots:
    void displayPicture(const QImage& picture);
    void saveAsPicture();

private:
    void initFilters();
    void updatePicturePixmap();

private:
    Ui::MainWindow *ui;
    QImage mSourcePicture;
    QImage mSourceThumbnail;
    QImage& mFilteredPicture;
    QPixmap mCurrentPixmap;

    FilterLoader mFilterLoader;
    FilterWidget* mCurrentFilter;
    QVector<FilterWidget*> mFilters;
};
```

唯一の注目すべき点は、メンバ変数として mFilterLoader が追加されたことです。MainWindow.cppでは、変更点のみを中心に説明します。

```C++
void MainWindow::initFilters()
{
    mFilterLoader.loadFilters();

    auto& filters = mFilterLoader.filters();
    for(auto& filter : filters) {
        FilterWidget* filterWidget = new FilterWidget(*filter);
        ui->filtersLayout->addWidget(filterWidget);
        connect(filterWidget, &FilterWidget::pictureProcessed,
            this, &MainWindow::displayPicture);
        mFilters.append(filterWidget);
    }

    if (mFilters.length() > 0) {
        mCurrentFilter = mFilters[0];
    }
}
```

initFilters() 関数は ui コンテンツからフィルタをロードしません。むしろ、mFilterLoader.loadFilters() 関数を呼び出して plugins ディレクトリからプラグインを動的にロードすることから始まります。

その後、mFilterLoader.filter()でauto&フィルターを割り当てています。auto キーワードを使った方がずっと読みやすいことに注意してください。本当の型は std::vector\<std::unique_ptr\<Filter\>\>& で、これは単純なオブジェクト型というよりは暗号の呪文のように見えます。

これらのフィルタのそれぞれについて、FilterWidget\* を作成して filter の参照を渡します。ここでは、filter は実質的に unique_ptr です。C++11 の背後にある人々は、参照元演算子を賢く修正して、新しい FilterWidget(*filter) から透過的になるようにしました。auto キーワードと -> 演算子のオーバーロード、つまり参照元演算子の組み合わせにより、C++ の新機能の使用がより楽しくなります。

for ループを見てください。それぞれの__filter__に対して、以下のタスクを行います。

1. FilterWidget テンプレートを作成します。
2. FilterWidget テンプレートを filtersLayout の子プロセスに追加します。
3. FilterWidget::pictureProcessedシグナルをMainWindow::displayPictureスロットに接続します。
4. 新しい FilterWidget テンプレートを QVectormFilters に追加します。

最終的には、最初のフィルタウィジェットが選択されます。

MainWindow.cpp の他の変更点は loadPicture() の実装だけです。

```C++
void MainWindow::loadPicture()
{
    ...
    for (int i = 0; i <mFilters.size(); ++i) {
    mFilters[i]->setSourcePicture(mSourcePicture);
    mFilters[i]->setSourceThumbnail(mSourceThumbnail);
    mFilters[i]->updateThumbnail();
    }
    mCurrentFilter->process();
}
```

updateThumbnail()関数は2つの関数に分割されていますが、ここではそれを利用しています。

これでアプリケーションのテストができるようになりました。実行してみると、動的プラグインが読み込まれ、処理されたデフォルトのレンナ画像が表示されているのが確認できるはずです。

***

**[戻る](../index.html)**
