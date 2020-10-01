# QSignalSpyでアプリケーションをスパイする

Qt Test フレームワークの最後の部分は、QSignalSpy を使ってシグナルをスパイする機能です。このクラスを使うと、任意の QObject の放出されたシグナルのイントロスペクションを行うことができます。

SoundEffectWidgetを使って実際に見てみましょう。SoundEffectWidget::play() 関数が呼び出されたときに、正しい soundId パラメータで soundPlayed シグナルが出ることをテストします。

TestGuiのplaySound()関数です。

```C++
#include <QTest>

#include "MainWindow.h"

// In TestGui.h
class TestGui : public QObject
{
    ...
    void controlButtonState();
    void playSound();
    ...
};

// In TestGui.cpp
#include <QPushButton>
#include <QtTest/QtTest>

#include "SoundEffectWidget.h"

...

void TestGui::playSound()
{
    SoundEffectWidget widget;
    QSignalSpy spy(&widget, &SoundEffectWidget::soundPlayed);
    widget.setId(2);
    widget.play();

    QCOMPARE(spy.count(), 1);
    QList<QVariant> arguments = spy.takeFirst();
    QCOMPARE(arguments.at(0).toInt(), 2);
}
```

SoundEffectWidget ウィジェットと QSignalSpy クラスを初期化することから始めます。spyクラスのコンストラクタは、スパイするオブジェクトへのポインタと、監視するシグナルのメンバ関数へのポインタを受け取ります。ここでは、SoundEffectWidget::soundPlayed()シグナルをチェックしたいと思います。

直後、widgetは任意のsoundId(2)で設定され、widget.play()が呼び出されます。ここからが面白いところです: spyはシグナルの放出されたパラメータをQVariantListに保存します。soundPlayed()が放出されるたびに、spyに新しいQVariantListが作成され、放出されたパラメータが格納されます。

まず、spy.count()を1と比較して、このシグナルが1回しか出ていないことを確認します。その直後に、このシグナルのパラメータをargumentsに格納し、widgetが設定した初期のsoundIdである値2を持っていることを確認します。

ご覧のように、QSignalSpyの使い方は簡単です。スパイしたい信号ごとに必要な数だけ作成することができます。

***

## まとめ

Qt Testモジュールは、テストアプリケーションを簡単に作成できるように優雅にサポートしてくれます。あなたは、スタンドアロンのテストアプリケーションでプロジェクトを整理することを学びました。単純なテストでは、特定の値を比較して検証することができます。複雑なテストでは、データセットを使用することができます。あなたは、関数を実行するために必要な時間やCPUティックを記録する、簡単なベンチマークを実装しました。アプリケーションが正常に動作することを確認するために、GUIイベントやQtシグナルのスパイをシミュレートしました。

アプリケーションが作成され、ユニットテストが PASS ステータスを示しています。次の章では、アプリケーションをデプロイする方法を学びます。

***

**[戻る](../index.html)**
