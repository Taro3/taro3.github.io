# コード責任の分散

これで、ユーザは作成時にタスク名を指定できるようになりました。名前を入力するときにエラーが発生したらどうしますか？論理的な手順はタスクを生成したあとにタスク名を変更することです。ここでは少し異なるアプローチをとります。Taskを可能な限り自律的にしたいのです。Taskを(MainWindowではなく)別のコンポーネントにアタッチしても、この名前変更の機能が使えなければなりません。そのために、Taskクラスに責任をもたせる必要があります。

```C++
// In Task.h
public slots:
    void rename();

// In Task.cpp
#include <QInputDialog>

Task::Task(const QString &name, QWidget *parent) :
    QWidget(parent),
    ui(new Ui::Task)
{
    ui->setupUi(this);
    setName(name);
    connect(ui->editButton, &QPushButton::clicked, this, &Task::rename);
}
...
void Task::rename()
{
    bool ok;
    QString value = QInputDialog::getText(this, tr("Edit task"),
                                          tr("Task name"),
                                          QLineEdit::Normal,
                                          this->name(), &ok);
    if (ok && !value.isEmpty()) {
        setName(value);
    }
}
```

シグナルに接続するために、パブリックスロットrename()を追加します。rename()は、以前にQInputDialogで説明した内容を再利用しています。唯一の違いは、QInputDialogのデフォルト値である現在のタスク名です。setName(value)が呼び出されると、UIは即座に新しい値で更新されます。

同期も更新も何もしなくても、Qtのメインループが仕事をしてくれます。MainWindowでは何も変更されていないので、Taskと親Widgetとの間の関連は事実上ありません。

***
**[戻る](../index.hrml)
