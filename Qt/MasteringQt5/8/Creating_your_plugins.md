# プラグインの作成

SDKの作成には苦労しませんでした。これで、最初のプラグインの作成に進むことができます。すべてのプラグインに先ほど完成させたSDKが含まれていることはすでにわかっています。幸いなことに、これは.priファイル（PRoject Include）で簡単に因数分解することができます。.pri ファイルは .pro ファイルと全く同じように動作します。唯一の違いは、.pro ファイルの中にインクルードすることを意図しているということです。

ch08-image-animation ディレクトリに plugins-common.pri という名前のファイルを作成し、その中に以下のコードを記述します。

```QMake
INCLUDEPATH += $$PWD/sdk
DEPENDPATH += $$PWD/sdk
```

このファイルは、各 .pro プラグインに含まれます。このファイルは、SDKのヘッダがどこにあるのか、ヘッダとソース間の依存関係を解決するためにどこを探せばよいのかをコンパイラに伝えることを目的としています。これにより、変更の検出が強化され、必要に応じて適切にソースをコンパイルすることができます。

このファイルをプロジェクトで見るためには、ch08-imageanimation.pro の OTHER_FILES マクロに追加する必要があります。

```QMake
OTHER_FILES += \
    sdk/Filter.h \
    plugins-common.pri
```

ビルドする最も簡単なプラグインは、イメージに対して特定の処理を実行しないため、filter-plugin-originalです。次の手順でこのプラグインを作成しましょう。

1. ch08-image-animationで新規**サブプロジェクト**を作成します。
2. 「**ライブラリ**」→「**C++ライブラリ**」→「**選択...**」を選択します。
3. **共有ライブラリ**を選択し、名前を filter-plugin-original とし、**Next** をクリックします。
4. **QtCore** を選択し、**QtWidgets** | **Next** を選択します。
5. 作成したクラス FilterOriginal に名前を付け、**Next** をクリックします。
6. 作成したクラスを ch08-image-animation の**サブプロジェクト**として追加し、**Finish** をクリックします。

Qt Creator は私たちのために多くのボイラプレートコードを作成してくれますが、今回の場合は必要ありません。filter-plugin-original.proをこのように更新します。

```QMake
QT += core widgets

TARGET = $$qtLibraryTarget(filter-plugin-original)
TEMPLATE = lib
CONFIG += plugin

SOURCES += \
    FilterOriginal.cpp

HEADERS += \
    FilterOriginal.h

include(../plugins-common.pri)
```

まず、$$qtLibraryTarget()で、TARGETの名前をOSの規約に従って適切に指定します。CONFIGプロパティにはpluginディレクティブが追加され、生成されたMakefileにDLL/SO/Dylibをコンパイルするために必要な命令を含めるように指示します（OSを選択してください）。

不要なDEFINESとFilterOriginal_global.hを削除しました。プラグインに固有のものは呼び出し元には何も公開されていないはずなので、シンボルのエクスポートを処理する必要はありません。

これで、FilterOriginal.hに進むことができます。

```C++
#include <QObject>
#include <Filter.h>

class FilterOriginal : public QObject, Filter
{
    Q_OBJECT
    Q_PLUGIN_METADATA(IID "org.masteringqt.imageanimation.filters.Filter")
    Q_INTERFACES(Filter)

public:
    FilterOriginal(QObject* parent = 0);
    ~FilterOriginal();
    QString name() const override;
    QImage process(const QImage& image) override;
};
```

FilterOriginal クラスは、まず QObject を継承しなければなりません。プラグインがロードされるときには、まず QObject クラスになってから実際の型である Filter にキャストされます。

Q_PLUGIN_METADATA マクロは、適切に実装されたインターフェイス識別子を Qt にエクスポートするように記述されています。これは、Qtメタシステムにクラスを知らせるための注釈を付けます。Filter.hで定義した一意の識別子に再び出会うことになります。

Q_INTERFACES マクロは、クラスが実装しているインターフェイスを Qt メタオブジェクトシステムに伝えます。

最後に、FilterOriginal.cppはかろうじて印刷するに値するものです。

```C++
FilterOriginal::FilterOriginal(QObject* parent) :
    QObject(parent)
{
}

FilterOriginal::~FilterOriginal()
{
}

QString FilterOriginal::name() const
{
    return "Original";
}

QImage FilterOriginal::process(const QImage& image)
{
    return image;
}
```

ご覧のように、その実装は何もしていません。第7章「頭の痛いことのないサードパーティライブラリ」からのバージョンに追加したのはname()関数だけで、これはOriginalを返します。

ここで，グレースケールフィルタを実装します。第7章「頭を痛めないサードパーティライブラリ」で行ったように、画像の処理はOpenCVライブラリに頼ることにします。以下のプラグインであるぼかしについても同じことが言えます。

これら2つのプロジェクトはそれぞれ独自の.proファイルを持っているので、OpenCVのリンクが同じになることはすでに予想できます。これは，.pri ファイルの完璧な使用例です。

ch08-image-animationディレクトリ内にplugins-commonopencv.priという新しいファイルを作成します。ch08-image-animation.pro内のOTHER_FILESに追加することを忘れないでください。

```QMake
OTHER_FILES += \
    sdk/Filter.h \
    plugins-common.pri \
    plugins-common-opencv.pri
```

plugins-common-opencv.priの内容です。

```QMake
windows {
    INCLUDEPATH += $$(OPENCV_HOME)/../../include
    LIBS += -L$$(OPENCV_HOME)/lib \
        -lopencv_core2413 \
        -lopencv_imgproc2413
}

linux {
    CONFIG += link_pkgconfig
    PKGCONFIG += opencv
}

macx {
    INCLUDEPATH += /usr/local/Cellar/opencv/2.4.13/include/
    LIBS += -L/usr/local/lib \
        -lopencv_core \
        -lopencv_imgproc
}
```

plugins-common-opencv.priの内容は、第7章「頭の痛いことのないサードパーティライブラリ」で作ったものをそのままコピーしたものです。

これですべての準備が整いました。これで filter-plugin-grayscale プロジェクトを進めることができます。filter-plugin-original と同様に、次のようにビルドします。

1. **共有ライブラリ**タイプのch08-image-animationの**C++ライブラリサブプロジェクト**を作成します。
2. FilterGrayscaleという名前のクラスを作成します。
3. **必要なモジュール**で、**QtCore**および**QWidgets**を選択します。

filter-plugin-grayscale.pro の更新版です。

```QMake
QT += core widgets

TARGET = $$qtLibraryTarget(filter-plugin-grayscale)
TEMPLATE = lib
CONFIG += plugin

SOURCES += \
    FilterGrayscale.cpp

HEADERS += \
    FilterGrayscale.h

include(../plugins-common.pri)
include(../plugins-common-opencv.pri)
```

内容は filter-plugin-original.pro と非常によく似ています。プラグインを OpenCV とリンクさせるために plugins-common-opencv.pri を追加しただけです。

FilterGrayscale については、ヘッダは FilterOriginal.h と全く同じです。以下は FilterGrayscale.cpp の関連部分です。

```C++
#include <opencv/cv.h>

// Constructor & Destructor here
...
QString FilterOriginal::name() const
{
    return "Grayscale";
}

QImage FilterOriginal::process(const QImage& image)
{
    // QImage => cv::mat
    cv::Mat tmp(image.height(),
        image.width(),
        CV_8UC4,
        (uchar*)image.bits(),
        image.bytesPerLine());

    cv::Mat resultMat;
    cv::cvtColor(tmp, resultMat, CV_BGR2GRAY);

    // cv::mat => QImage
    QImage resultImage((const uchar *) resultMat.data,
        resultMat.cols,
        resultMat.rows,
        resultMat.step,
        QImage::Format_Grayscale8);
    return resultImage.copy();
}
```

plugins-common-opencv.pri を含めることで、 cv.h ヘッダを適切に含めることができます。

最後に実装するプラグインはブラープラグインです。もう一度、C++ライブラリサブプロジェクトを作成し、FilterBlurクラスを作成します。プロジェクトの構造と.proファイルの内容は同じです。こちらがFilterBlur.cppです。

```C++
QString FilterOriginal::name() const
{
    return "Blur";
}

QImage FilterOriginal::process(const QImage& image)
{
    // QImage => cv::mat
    cv::Mat tmp(image.height(),
        image.width(),
        CV_8UC4,
        (uchar*)image.bits(),
        image.bytesPerLine());

    int blur = 17;
    cv::Mat resultMat;
    cv::GaussianBlur(tmp,
        resultMat,
        cv::Size(blur, blur),
        0.0,
        0.0);

    // cv::mat => QImage
    QImage resultImage((const uchar *) resultMat.data,
        resultMat.cols,
        resultMat.rows,
        resultMat.step,
        QImage::Format_RGB32);
    return resultImage.copy();
}
```

ボケ量は17とハードコードされています。本番のアプリケーションでは、この量をアプリケーションから可変にするのが説得力があったかもしれません。

***

## Tips

プロジェクトをさらに進めたいのであれば、プラグインのプロパティを設定する方法を含むレイアウトをSDKに含めてみてください。

***

**[戻る](../index.html)**
