# auto型とレンジベースのforループでシンプルに

タスクを完全にCRUDにするための最後のステップは、完了したタスクの機能を実装することです。
以下のように実装します。

* チェックボックスをクリックしてタスクを完了としてマークする
* タスク名を打ち消し文字状態にする
* MainWindowのステータスラベルを更新する

チェックボックスのクリック処理は、removedと同じパターンに従います。

```C++
// In Task.h
signals:
    void removed(Task* task);
    void statusChanged(Task* task);

private slots:
    void checked(bool checked);

// In Task.cpp
Task::Task(const QString &name, QWidget *parent) :
    QWidget(parent),
    ui(new Ui::Task)
{
    ...

    connect(ui->checkbox, &QCheckBox::toggled,
            this, &Task::checked);
}

...

void Task::checked(bool checked)
{
    QFont font(ui->checkbox->font());
    font.setStrikeOut(checked);
    ui->checkbox->setFont(font);
    emit statusChanged(this);
}
```

checkbox::toggledシグナルに接続するスロットchecked(bool checked)を定義しています。slot checked()の中で、bool checkedの値に応じてcheckboxのテキストを打ち消し文字にしています。これはQFontクラスを使って行います。checkbox->font()からコピーフォントを作成し、それを修正してui->checkboxに戻します。元のfontが太字で特別なサイズだった場合は、その外観は変わらないことが保証されています。

## Tip

Qt Designerでフォントオブジェクトを弄ってみましょう。Task.uiファイルのチェックボックスを選択し、プロパティエディタ|QWidget|fontで行います。

最後の命令は、Taskのステータスが変更されたことをMainWindowに通知します。タスクの実装の詳細を隠すために、シグナル名をcheckboxCheckedではなくstatusChangedにしています。MainWindow.hファイルに以下のコードを追加します。

```C++
// In MainWindow.h
public:
    MainWindow(QWidget *parent = nullptr);
    ~MainWindow();
    void updateStatus();

public slots:
    void addTask();
    void removeTask(Task* task);
    void taskStatusChanged(Task* task);

// In MainWindow.cpp
MainWindow::MainWindow(QWidget *parent)
    : QMainWindow(parent)
    , ui(new Ui::MainWindow)
    , mTasks()
{
    ...
    updateStatus();
}

void MainWindow::addTask()
{
    ...
    if (ok && !name.isEmpty()) {
        ...
        connect(task, &Task::removed,
                this, &MainWindow::removeTask);
        connect(task, &Task::statusChanged, this,
                &MainWindow::taskStatusChanged);
        mTasks.append(task);
        ui->tasksLayout->addWidget(task);
        updateStatus();
    }
}

void MainWindow::removeTask(Task *task)
{
    ...
    delete task;
    updateStatus();
}

void MainWindow::taskStatusChanged(Task* /*task*/)
{
    updateStatus();
}

void MainWindow::updateStatus()
{
    int completedCount = 0;
    for (auto t : mTasks) {
        if (t->isCompleted()) {
            ++completedCount;
        }
    }
    int todoCount = mTasks.size() - completedCount;

    ui->statusLabel->setText(
                QString("Status: %1 todo / %2 completed")
                .arg(todoCount)
                .arg(completedCount));
}
```

タスクが作成されたときに接続されるスロットtaskStatusChangedを定義しました。このスロットの唯一の仕事はupdateStatus()を呼び出すことです。この関数はタスクを反復処理し、statusLabelを更新します。updateStatus()関数はタスクの作成時と削除時に呼び出されます。

updateStatus()では、以下のような新しいC++11のセマンティクスが追加されています。

```C++
for(auto t : mTasks) {
    ...
}
```

forキーワードを使用すると範囲ベースのコンテナをループさせることができます。QVectorは反復可能なコンテナなので、ここではそれを使うことができます。範囲宣言(auto t)は、各反復時に代入される型と変数名です。範囲式(mTask)は、単純に処理が行われるコンテナです。Qtは、以前のバージョンのC++を対象としたfor(つまりforeach)ループのカスタム実装を提供しています。

autoキーワードはもう一つの素晴らしい新しいセマンティクスです。コンパイラはイニシャライザに基づいて変数の型を自動的に推論します。これにより、以下のような暗号のようなイテレータに対する多くの苦情が軽減されます。

```C++
    std::vector::const_iterator iterator = mTasks.toStdVector()
                                                .stdTasks.begin();

    // how many neurones did you save?
    auto autoIter = stdTasks.begin();
```

C++14以降、autoは関数の戻り値型にも使用できるようになりました。これは素晴らしいツールですが、使用は控えめにしてください。autoを使用する場合は、シグネチャ名/変数名から型が明らかである必要があります。

## Tip

autoキーワードは、このようにconstや参照と組み合わせることができます。
for (const auto & t : mTasks) { ... }

先程のラムダ式を覚えていますか？すべての機能が使えるので、次のように書くことができます。

```C++
    auto prettyName = [] (const QString& taskName) -> QString {
        return "-------- " + taskName.toUpper();
    };
    connect(ui->removeButton, &QPushButton::clicked,
        [this, name, prettyName] {
            qDebug() << "Trying to remove" << prettyName(name);
            this->emit removed(this);
        });
```

これは素晴らしいことです。autoとラムダを組み合わせることで、非常に読みやすいコードになり、可能性の世界が広がります。

最後の勉強項目はQString APIです。updateStatus()で使いました。

```C++
    ui->statusLabel->setText(
        QString("Status: %1 todo / %2 completed")
        .arg(todoCount)
        .arg(completedCount));
```

Qtを作った人たちはC++で文字列操作を簡単に行うために多くの作業を行いました。
これは完璧な例であり、古典的なCのsprintfをモダンで堅牢なAPIに置き換えます。引数はポジションベースのみで、型を指定する必要がなく(エラーが起こりにくい)、arg(...)関数はすべての種類の型を受け入れます。

## Tip

少し時間を取って、http://doc.qt.io/qt-5/qstring.html にあるQStringのドキュメントをざっと見てください。これは、このクラスでどれだけのことができるかを示しており、std stringやcstringを使うことは少なくなるでしょう。

***
**[戻る](../index.html)**
