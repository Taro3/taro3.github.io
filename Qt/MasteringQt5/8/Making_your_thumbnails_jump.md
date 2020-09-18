# サムネイルをジャンプさせる

Qtアニメーションフレームワークについて学んだことをプロジェクトに適用してみましょう。ユーザーがフィルタのサムネイルをクリックするたびに、それを突いてみたいと思います。すべての変更は FilterWidget クラスで行います。まずは FilterWidget.h から始めましょう。

```C++
#include <QPropertyAnimation>

class FilterWidget : public QWidget
{
    Q_OBJECT

public:
    explicit FilterWidget(Filter& filter, QWidget *parent = 0);
    ~FilterWidget();
    ...

private:
    void initAnimations();
    void startSelectionAnimation();

private:
    ...
    QPropertyAnimation mSelectionAnimation;
};
```

最初の関数 initAnimations() は、FilterWidget が使用するアニメーションを初期化します。2 番目の関数 startSelectionAnimation() は、このアニメーションを正しく開始するために必要なタスクを実行します。ご覧のように、前のセクションで説明したように QPropertyAnimation クラスも宣言しています。

これで FilterWidget.cpp を更新することができます。コンストラクタを更新してみましょう。

```C++
FilterWidget::FilterWidget(Filter& filter, QWidget *parent) :
    QWidget(parent),
    ...
    mSelectionAnimation()
{
    ...
    initAnimations();
    updateThumbnail();
}
```

mSelectionAnimationと呼ばれるQPropertyAnimationを初期化します。コンストラクタは initAnimations() も呼び出します。以下にその実装を示します。

```C++
void FilterWidget::initAnimations()
{
    mSelectionAnimation.setTargetObject(ui->thumbnailLabel);
    mSelectionAnimation.setPropertyName("geometry");
    mSelectionAnimation.setDuration(200);
}
```

これでアニメーションの初期化の手順はお分かりになったと思います。対象となるオブジェクトは、フィルタプラグインのプレビューを表示しているサムネイルラベルです。このQLabelの位置を更新したいので、アニメーションさせるプロパティ名はgeometryとします。最後に、アニメーションの持続時間を 200 ms に設定します。ジョークのように、短くて甘いものにしましょう。

既存のマウスイベントハンドラを次のように更新します。

```C++
void FilterWidget::mousePressEvent(QMouseEvent*)
{
    process();
    startSelectionAnimation();
}
```

ユーザーがサムネイルをクリックするたびに、サムネイルを動かす選択アニメーションが呼び出されます。この最も重要な機能をこのように追加することができるようになりました。

```C++
void FilterWidget::startSelectionAnimation()
{
    if (mSelectionAnimation.state() ==
        QAbstractAnimation::Stopped) {
        QRect currentGeometry = ui->thumbnailLabel->geometry();
        QRect targetGeometry = ui->thumbnailLabel->geometry();
        targetGeometry.setY(targetGeometry.y() - 50.0);
        mSelectionAnimation.setKeyValueAt(0, currentGeometry);
        mSelectionAnimation.setKeyValueAt(0.3, targetGeometry);
        mSelectionAnimation.setKeyValueAt(1, currentGeometry);
        mSelectionAnimation.start();
    }
}
```

まず最初に、currentGeometryというサムネイルラベルの現在のジオメトリを取得します。次に、同じx、width、heightの値を持つtargetGeometryオブジェクトを作成します。yの位置を50下げるだけなので、ターゲットの位置は常に現在の位置より上になります。

その後、キーフレームを定義します。

* **ステップ0**では、現在位置が値となります。
* **ステップ0.3**(60ミリ秒、合計200ミリ秒なので)では、その値が目標位置となる。
* **ステップ1**（アニメーションの終了）では、元の位置に戻します。サムネイルはすぐに目標位置に到達し、その後ゆっくりと元の位置に落ちていきます。

これらのキーフレームは、各アニメーションの開始前に初期化する必要があります。レイアウトは動的なので、ユーザーがメインウィンドウのサイズを変更したときに位置（そしてジオメトリ）が更新される可能性があります。

現在の状態が停止していないと、再びアニメーションが開始されないように予防していますので、ご注意ください。この予防策がないと、ユーザーがウィジェット上で狂人のようにクリックすると、サムネイルが何度も上に移動してしまう可能性があります。

これでアプリをテストして、フィルター効果をクリックすることができるようになりました。フィルターのサムネイルがクリックに反応してジャンプします!

***

**[戻る](../index.html)**
