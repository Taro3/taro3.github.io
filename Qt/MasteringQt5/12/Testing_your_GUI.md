# GUIのテスト

Qt Test APIを使ってGUIをテストする方法を見ていきましょう。QTestクラスには、キーやマウスのイベントをシミュレートするための関数がいくつか用意されています。

実証するために、Track の状態をテストするという概念はそのままに、より上位のレベルでテストしてみましょう。Track の状態そのものをテストするのではなく、Track の状態が変更されたときにdrum-machineアプリケーションの UI の状態が適切に更新されているかどうかをチェックします。つまり、録音が開始されたときにコントロールボタン(play, stop, record)が特定の状態になっている必要があります。

ドラムマシンテストプロジェクトにTestGuiクラスを作成することから始めます。main.cppのテストマップにTestGuiクラスを追加するのを忘れずに。いつものようにQObjectを継承させてTestGui.hを更新します。

```C++
#include <QTest>

#include "MainWindow.h"

class TestGui : public QObject
{
    Q_OBJECT
public:
    TestGui(QObject* parent = 0);

private:
    MainWindow mMainWindow;
};
```

このヘッダには、drum-machineプロジェクトのMainWindowキーワードのインスタンスであるmMainWindowというメンバがあります。TestGuiのテストでは、1つのMainWindowが使用され、イベントを注入して反応をチェックします。

TestGuiのコンストラクタに切り替えてみましょう。

```C++
#include <QtTest/QtTest>

TestGui::TestGui(QObject* parent) :
    QObject(parent),
    mMainWindow()
{
    QTestEventLoop::instance().enterLoop(1);
}
```

コンストラクタは mMainWindow 変数を初期化します。mMainWindowは決して表示されないことに注意してください (mMainWindow.show() を使用しています)。表示する必要はなく、状態をテストしたいだけなのです。

ここでは、1秒後にイベントループを強制的に開始するために、かなり曖昧な関数呼び出しを使用しています（QTestEventLoopは全く文書化されていません）。

なぜこのようなことをしなければならないかというと、QSoundEffect クラスにあります。QSoundEffectクラスは、QSoundEffect::setSource()関数が呼ばれたときに初期化されます（MainWindowでは、SoundEffectWidgetsの初期化時に行われます）。明示的な enterLoop() の呼び出しを省略すると、drum-machine-test の実行はセグメンテーション障害でクラッシュします。

QSoundEffectクラスが適切に初期化を完了するためには、イベントループを明示的に入力しなければならないようです。QSoundEffectクラスのQtユニットテストを研究することで、この文書化されていない回避策を発見しました。

では、実際のGUIテストをしてみましょう!コントロールボタンをテストするには、TestGuiをアップデートします。

```C++
// In TestGui.h
class TestGui : public QObject
{
    ...
private slots:
    void controlButtonState();
    ...
};

// In TestGui.cpp
#include <QtTest/QtTest>
#include <QPushButton>

...

void TestGui::controlButtonState()
{
    QPushButton* stopButton =
    mMainWindow.findChild<QPushButton*>("stopButton");
    QPushButton* playButton =
    mMainWindow.findChild<QPushButton*>("playButton");
    QPushButton* recordButton =
    mMainWindow.findChild<QPushButton*>("recordButton");

    QTest::mouseClick(recordButton, Qt::LeftButton);

    QCOMPARE(stopButton->isEnabled(), true);
    QCOMPARE(playButton->isEnabled(), false);
    QCOMPARE(recordButton->isEnabled(), false);
}
```

controlButtonState() 関数では、まず、便利な mMainWindow.findChild() 関数を使ってボタンを取得します。この関数は QObject で利用可能で、渡された名前は、MainWindow.ui を作成した際に Qt Designer で各ボタンに使用した objectName 変数に対応しています。

すべてのボタンを取得したら、QTest::mouseClick()関数を使用してマウスクリックイベントを発生させます。これは QWidget* パラメータをターゲットとし、クリックするボタンを指定します。キーボード修飾子 (control や shift など) とクリックの遅延時間 (ミリ秒単位) を渡すこともできます。

***

## Info

この機能は、入力が目的の状態、出力が期待されるボタンの状態であるデータセットを用いて、すべての状態（PLAYING, STOPPED, RECORDING）をテストするために簡単に拡張することができます。

***

QTestクラスには、以下のようなイベントを注入するための多くの便利な関数があります。

* keyEvent(): この関数は、キーイベントをシミュレートするために使用されます。
* keyPress(): この関数は、キープレスイベントをシミュレートするために使用されます。
* keyRelease(): この関数は、キーリリースイベントをシミュレートするために使用されます。
* mouseClick(): この関数は、キークリックイベントをシミュレートするために使用します。
* mouseDClick(): この関数は、マウスのダブルクリックイベントをシミュレートするために使用します。
* mouseMove(): この関数は、マウスの移動イベントをシミュレートするために使用します。

***

**[戻る](../index.html)**
