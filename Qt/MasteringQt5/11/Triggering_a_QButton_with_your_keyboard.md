# キーボードでQButtonをトリガーする

SoundEffectWidget クラスのパブリックスロット、triggerPlayButton() を調べてみましょう。

```C++
//SoundEffectWidget.h
class SoundEffectWidget : public QWidget
{
...
public slots:
    void triggerPlayButton();
    ...

private:
    QPushButton* mPlayButton;
    ...
};

//SoundEffectWidget.cpp
void SoundEffectWidget::triggerPlayButton()
{
    mPlayButton->animateClick();
}
```

このウィジェットには mPlayButton という QPushButton があります。triggerPlayButton（）スロットはQPushButton :: animateClick（）関数を呼び出します。この関数は、デフォルトで100ミリ秒を超えるボタンのクリックをシミュレートします。すべてのシグナルは、実際のクリックと同じように送信されます。ボタンは本当に下に落ちているように見えます。アニメーションが不要な場合は、QPushButton::click() を呼び出すことができます。

それでは、このスロットをキーでトリガーする方法を見てみましょう。各 SoundEffectWidget には Qt:Key があります。

```C++
//SoundEffectWidget.h
class SoundEffectWidget : public QWidget
{
...
public:
    Qt::Key triggerKey() const;
    void setTriggerKey(const Qt::Key& triggerKey);
};

//SoundEffectWidget.cpp
Qt::Key SoundEffectWidget::triggerKey() const
{
    return mTriggerKey;
}
void SoundEffectWidget::setTriggerKey(const Qt::Key& triggerKey)
{
    mTriggerKey = triggerKey;
}
```

SoundEffectWidgetクラスは、メンバー変数mTriggerKeyを取得・設定するゲッターとセッターを提供します。

MainWindowクラスは、4つのSoundEffectWidgetのキーを以下のように初期化します。

```C++
ui->kickWidget->setTriggerKey(Qt::Key_H);
ui->snareWidget->setTriggerKey(Qt::Key_J);
ui->hihatWidget->setTriggerKey(Qt::Key_K);
ui->crashWidget->setTriggerKey(Qt::Key_L);
```

デフォルトでは、QObject::eventFilter() 関数は呼び出されません。これを有効にしてこれらのイベントを傍受するには、MainWindow にイベントフィルタをインストールする必要があります。

```C++
installEventFilter(this);
```

そのため、MainWindow がイベントを受信するたびに MainWindow::eventFilter() 関数が呼び出されます。

ここにMainWindow.hヘッダがあります。

```C++
class MainWindow : public QMainWindow
{
    Q_OBJECT

public:
    ...
    bool eventFilter(QObject* watched, QEvent* event) override;

private:
    QVector<SoundEffectWidget*> mSoundEffectWidgets;
    ...
};
```

MainWindowクラスは、4つのSoundEffectWidgets (kickWidget, snareWidget, hihatWidget, crashWidget)を持つQVectorを持っています。MainWindow.cppでの実装を見てみましょう。

```C++
bool MainWindow::eventFilter(QObject* watched, QEvent* event)
{
    if (event->type() == QEvent::KeyPress) {
        QKeyEvent* keyEvent = static_cast<QKeyEvent*>(event);
        for(SoundEffectWidget* widget : mSoundEffectWidgets) {
            if (keyEvent->key() == widget->triggerKey()) {
                widget->triggerPlayButton();
                return true;
            }
        }
    }
    return QObject::eventFilter(watched, event);
}
```

まず、QEventクラスがKeyPress型であることを確認します。他のイベントタイプは気にしません。イベントタイプが正しい場合は、以下の手順に進みます。

1. QEvent クラスを QKeyEvent にキャストします。
2. そして、押されたキーがSoundEffectWidgetクラスに属しているかどうかを検索します。
3. キーに対応する SoundEffectWidget クラスがあれば、SoundEffectWidget::triggerPlayButton() 関数を呼び出し、イベントを消費したことを示す true を返し、他のクラスに伝播してはいけないことを示します。
4. それ以外の場合は、QObject クラスの eventFilter() の実装を呼び出します。

***

**[戻る](../index.html)**
