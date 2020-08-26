# ラムダ式を使用したカスタムシグナルのエミット

タスク削除の実装は簡単ですが、途中でいくつかの新しい概念を勉強してみましょう。Taskは、所有者と親(MainWindow)にremoveTaskButtonのQPushButtonがクリックされたことを通知しなければなりません。Task.hファイルにカスタムシグナルremovedを定義して実装します。

```C++
class Task : public QWidget
{
    ...
public slots:
    void rename();

signals:
    void removed(Task* task);
    ...
};
```

スロットで行ったように、ベッダにQtキーワードのsignalsを追加する必要があります。シグナルは他のクラスに通知するためだけに使われるので、publicキーワードは必要ありません(コンパイルエラーになります)。シグナルは単にレシーバ(接続されたスロット)に送られる通知であり、removed(Task* task)関数の関数本体がないことを意味します。レシーバがどのタスクが削除を要求したかを知ることができるように、taskパラメータを追加しました。次のステップは、removeButtonがクリックされたときにremovedシグナルを出すことです。これはTask.cppファイルで行います。

```C++
Task::Task(const QString &name, QWidget *parent) :
    QWidget(parent),
    ui(new Ui::Task)
{
    ui->setupUi(this);
    ...
    connect(ui->editButton, &QPushButton::clicked, [this] {
        emit removed(this);
    });
}
```

このコードの抜粋は、C++11の非常に興味深い機能であるラムダ式を示しています。この例では、ラムダ式は次の部分です。

```C++
[this] {
    emit removed(this);
});
```

ここで行ったのは、クリックされたシグナルを匿名のインライン関数であるラムダ式に接続することです。Qtは、シグネチャが一致する場合にシグナルを別のシグナルに接続することにより、シグナルの中継を可能にしますが、ここでは適用できません。clickedシグナルにはパラメータがなく、removedシグナルにはTask*が必要です。ラムダ式はTaskでの冗長なスロット宣言を不要にします。Qt 5はconnectのスロットの代わりにラムダ式を受け入れることができ、両方の構文を使用できます。

ラムダ式はemit removed(this)という1行のコードを実行します。emitはQtマクロであり、パラメータで渡したものを使用して接続されたslotをすぐにトリガします。前に述べたように、removed(Task* this)には関数本体がありません。その目的は、登録されたスロットにイベントを通知することです。

ラムダ式はC++へのすばらしい追加機能です。これはコードで短い関数を定義する非常に実用的な方法を提供します。技術的には、ラムダ式はスコープ内の変数をキャプチャできるクロージャの構築です。
完全な構文は次のようになります。

```C++
[capture-list] (params)->ret { body }...
```

このステートメントの各部分を調べてみましょう。

* **capture-list**: これは、ラムダ式のスコープ内で可視化される変数を定義します。
* **params**: これは、ラムダ式のスコープに渡す事ができる関数パラメータの型リストです。この場合、パラメータはありません。[this] () { ... }とも書けますが、C++11では括弧を省略できます。
* **ret**: これはラムダ関数の戻り値型です。paramsと同様に、戻り地の型がvoidの場合、このパラメータは省略できます。
* **body**: これはcapture-listとparamsにアクセスできるコードの本体であり、ret型の変数を返す必要があります。

この例では、thisポインタをキャプチャして実行できるようにしました。
Taskクラスの一部であるremoved()関数への参照があるので、thisをキャプチャしなかった場合、コンパイラは、error: 'this' was not captured for this lambda function emit removed(this);のように出力するでしょう。

thisをremovedシグナルに渡します(呼び出し元はどのタスクがremovedをトリガしたかを知る必要があります)。

capture-listは、標準のC++セマンティクスに依存しています。コピーまたは参照によって変数をキャプチャします。コンストラクタパラメータnameのログを出力したいとします。

```C++
connect(ui-removeButton, &QPushButton::clicked, [this, &name] {
    qDebug() << "Trying to remove" << name;
    this->emit removed(this);
});
```

このコードは正常にコンパイルされます。残念ながらTaskを削除しようとすると見事なセグメンテーション違反でクラッシュします。なぜでしょうか？前述したように、ラムダ式は、clicked()シグナルがエミットされたときに実行される匿名関数です。nameの参照をキャプチャしたので、この参照はTaskコンストラクタから(より正確には、呼び出し元のスコープから)出ると無効になる可能性があります。
qDebug()関数は、到達不能なコードを表示しようとしてクラッシュするでしょう。

キャプチャする対象とラムダ式が実行されるコンテキストには注意が必要です。この例では、コピーによってnameをキャプチャすることでセグメンテーション違反を修正できます。

```C++
connect(ui->removeButton, &QPushButton::clicked, [this, name] {
    qDebug() << "Trying to remove" << name;
    this->emit remove(this);
});
```

## C++ tip

* 構文[=]と[&]を使用してラムダ式を定義する関数で到達可能なすべての変数をコピーまたは参照してキャプチャできます。
* this変数は、キャプチャリストの特殊なケースです。参照[&this]でキャプチャすることはできません。このように指定した場合、コンパイラは警告を出力します([=, this] Don't do this.)

ラムダ式はパラメータとして直接connect関数に渡されます。つまり、ラムダ式は変数です。これには多くの影響があります。それを呼び出したり、代入したり、返したりすることができます。「完全な形の」ラムダ式を示すために、タスク名がフォーマットされたバージョンを返すラムダ式を定義することができます。このコードの唯一の目的は、ラムダ関数の仕組みを調べることです。次のコードをtodoアプリに含まないようにしてください。

```C++
connect(ui->removeButton, &QPushButton::clicked, [this, name] {
    qDebug() << "Trying to remove" <<
    [] (const QString& taskName) -> QString {
        return "-------- " + taskName.toUpper();
    }(name);
    this->emit removed(this);
});
```

ここではトリッキーなことを行いました。qDebug()の呼び出しの中で、即実行されるラムダ式を定義しました。分析してみましょう。

* **[]**: キャプチャは行いません。ラムダ式はそれを囲む関数に依存しません。
* **(const QString& taskName)**: このラムダ式が呼び出されると、QStringを処理することを期待します。
* **-> QString**: ラムダ式の戻り値はQStringになります。
* **return "-------- " + taskName.toUpper()**: ラムダ式の本体。文字列とパラメータtaskNameの大文字バージョンを連結したものを返します。Qtを使うと文字列の操作がとても簡単になります。
* **(name)**: ここで違和感があります。ラムダ関数が定義されたので、nameパラメータを渡して呼び出すことができます。1つの命令でそれを定義して呼び出すのです。QDebug()関数は単に結果を表示します。

このラムダ式を変数に代入して複数回呼び出すことができれば、このラムダの真価が発揮されます。C++は静的型付けされているので、ラムダ変数の型を指定しなければなりません。
言語仕様では、ラムダ式の型を明示的に定義することはできません。C++11でどのように処理するかはすぐに分かります。とりあえず、削除機能を終わらせましょう。

Taskはこれで、remove()シグナルをエミットします。このシグナルはMainWindowによって使用する必要があります。

```C++
// In MainWindow.h
public slots:
    void addTask();
    void removeTask(Task* task);

// In MainWindow.cpp
void MainWindow::addTask()
{
    ...
    if (ok && !name.isEmpty()) {
        qDebug() << "Adding new task";
        Task* task = new Task(name);
        connect(task, &Task::removed,
                this, &MainWindow::removeTask);
        ...
    }
}

void MainWindow::removeTask(Task *task)
{
    mTasks.removeOne(task);
    ui->tasksLayout->removeWidget(task);
    task->setParent(nullptr);
    delete task;
}

```

MainWindow::removeTask()はシグナルシグネチャと一致している必要があります。タスクが生成されたときにconnectが行われます。興味深いのはMainWindow::removeTask()の実装です。

タスクはまずmTasksベクタから削除されます。その後、tasksLayoutから削除されます。
ここで、tasksLayoutはtaskの所有権を開放します(つまり、taskLayoutはtaskクラスの親ではなくなります)。

ここまでは順調です。次の2行が興味深いのです。所有権の移転では、taskクラスの所有権が完全に開放されるわけではありません。この行をコメントにすると、removeTask()は次のようになります。

```C++
void MainWindow::removeTask(Task* task)
{
    mTasks.removeOne(task);
    ui->tasksLayout->removeWidget(task);
    // task->setParent(0);
    // delete task;
}
```

Taskのデストラクタにログメッセージを追加してプログラムを実行した場合、そのログメッセージは表示されます。にもかかわらず、QtドキュメントのQLayout::removeWidgetの部分では、ウィジェットの所有権は追加されたときのままです。

代わりに、実際に何が起こるかというと、taskクラスの親はtasksLayoutクラスの親であるcentralWidgetになります。Qtはtaskのことをすべて忘れてほしいので、task->setParent(nullptr)を呼び出しています。これで安全に削除することができます。

***
**[戻る](../index.html)**
